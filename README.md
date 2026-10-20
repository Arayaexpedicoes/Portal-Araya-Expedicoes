# Portal da Equipe Araya · robô que copia as fichas do Notion para o portal.
# Roda sozinho a cada hora e também pelo botão "Run workflow" (aba Actions).
# As chaves ficam nos segredos do GitHub (NOTION_TOKEN e ROBO_SENHA),
# nunca neste arquivo. O registro da execução mostra só contagens, sem conteúdo.
name: Sincronizar fichas do Notion

on:
  schedule:
    - cron: "17 * * * *"
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: sincronizar-notion
  cancel-in-progress: false

jobs:
  sincronizar:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Preparar
        run: npm install --no-save --no-audit --no-fund --loglevel=error firebase@10.14.1

      - name: Copiar fichas do Notion para o portal
        env:
          NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
          ROBO_SENHA: ${{ secrets.ROBO_SENHA }}
          ROBO_EMAIL: robo.portal@arayaexpedicoes.com
          NOTION_DATABASE_ID: ead044fa4f87493b86621b16719cf5e8
          NOTION_DATA_SOURCE_ID: e2f3a87a2c084e83906e5393dbbab421
        run: |
          cat > sincronizar.cjs <<'FIM_DO_ROBO'
          // Portal da Equipe Araya · robô Notion → Firestore
          // Lê as fichas com Status = Publicado e grava destinos/passeios no Firestore.
          // Não imprime nenhum conteúdo das fichas: só contagens.
          const { initializeApp } = require("firebase/app");
          const { getAuth, signInWithEmailAndPassword } = require("firebase/auth");
          const { getFirestore, collection, doc, getDocs, writeBatch, terminate } = require("firebase/firestore");

          // Configuração pública do portal (a mesma que está no index.html; não é segredo).
          const FIREBASE_CONFIG = {
            apiKey: "AIzaSyAUk1EYAL2QPk9IjnoE0Qq0LWbjtkdDx0o",
            authDomain: "portal-araya-expedicoes.firebaseapp.com",
            projectId: "portal-araya-expedicoes",
          };

          const NOTION_VERSION = "2022-06-28";
          const DB_ID = (process.env.NOTION_DATABASE_ID || "").replace(/-/g, "");
          const DS_ID = (process.env.NOTION_DATA_SOURCE_ID || "").replace(/-/g, "");

          const DESTINOS = {
            "Cusco e Machu Picchu": ["cusco-machu-picchu", 1],
            "Patagônia": ["patagonia", 2],
            "Japão": ["japao", 3],
            "Egito e Jordânia": ["egito-jordania", 4],
            "Ushuaia": ["ushuaia", 5],
            "Mendoza": ["mendoza", 6],
            "Atacama e Uyuni": ["atacama-uyuni", 7],
          };
          const SECOES = {
            "descricao": "descricao", "o que inclui": "inclui", "inclui": "inclui",
            "caracteristicas e atencao": "atencao", "atencao": "atencao",
            "gastronomia": "gastronomia", "guia interno": "guia",
          };
          const norm = (s) => String(s || "").normalize("NFD").replace(/[̀-ͯ]/g, "").toLowerCase().replace(/[^a-z0-9 ]+/g, " ").replace(/\s+/g, " ").trim();
          const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

          class Falha extends Error { constructor(m, status) { super(m); this.status = status; } }

          async function notion(path, body, versao) {
            for (let tentativa = 1; ; tentativa++) {
              const res = await fetch("https://api.notion.com/v1/" + path, {
                method: body ? "POST" : "GET",
                headers: {
                  "Authorization": "Bearer " + process.env.NOTION_TOKEN,
                  "Notion-Version": versao || NOTION_VERSION,
                  "Content-Type": "application/json",
                },
                body: body ? JSON.stringify(body) : undefined,
              });
              if (res.ok) return res.json();
              if ((res.status === 429 || res.status >= 500) && tentativa < 6) {
                const espera = Number(res.headers.get("retry-after")) || tentativa * 2;
                await sleep(espera * 1000);
                continue;
              }
              let codigo = "";
              try { codigo = (await res.json()).code || ""; } catch (_) {}
              if (res.status === 401) throw new Falha("O Notion recusou a chave (NOTION_TOKEN). Confira se o segredo foi colado inteiro.");
              if (res.status === 404 || codigo === "object_not_found") throw new Falha("O Notion não encontrou o banco de fichas. Confira se a conexão “Portal Araya” foi adicionada ao banco “Fichas de Passeios · Portal Araya” (••• → Conexões).", 404);
              throw new Falha("O Notion respondeu com erro " + res.status + (codigo ? " (" + codigo + ")" : "") + ".");
            }
          }

          // ---------- propriedades ----------
          const texto = (rt) => (rt || []).map((t) => t.plain_text || "").join("").trim();
          function prop(p) {
            if (!p) return null;
            switch (p.type) {
              case "title": return texto(p.title);
              case "rich_text": return texto(p.rich_text);
              case "select": return p.select ? p.select.name : "";
              case "status": return p.status ? p.status.name : "";
              case "multi_select": return (p.multi_select || []).map((o) => o.name);
              case "number": return p.number;
              case "date": return p.date ? p.date.start : "";
              case "people": return (p.people || []).map((u) => u.name || "").filter(Boolean);
              case "last_edited_time": return p.last_edited_time;
              default: return null;
            }
          }

          // ---------- conteúdo da página → markdown simples por seção ----------
          function inline(rt) {
            let out = "";
            for (const t of rt || []) {
              let s = (t.plain_text || "").replace(/\n/g, " ");
              if (!s) continue;
              const href = t.href && /^https?:\/\//.test(t.href) ? t.href : "";
              const negrito = t.annotations && t.annotations.bold && s.trim();
              if (negrito) {
                const ini = s.match(/^\s*/)[0], fim = s.match(/\s*$/)[0];
                s = ini + "**" + s.trim() + "**" + fim;
              }
              if (href && s.trim()) s = "[" + s.trim() + "](" + href + ")";
              out += s;
            }
            return out.replace(/\*\*\*\*/g, "").trim();
          }

          async function filhos(id) {
            const todos = [];
            let cursor;
            do {
              const r = await notion("blocks/" + id + "/children?page_size=100" + (cursor ? "&start_cursor=" + cursor : ""));
              todos.push(...r.results);
              cursor = r.has_more ? r.next_cursor : null;
            } while (cursor);
            return todos;
          }

          async function lerPagina(pageId) {
            const ficha = { secoes: {}, midias: 0, extras: [] };
            let atual = "descricao";
            const add = (linha) => { (ficha.secoes[atual] = ficha.secoes[atual] || []).push(linha); };

            async function percorrer(blocos, nivel) {
              const recuo = "  ".repeat(nivel);
              let n = 0;
              for (const b of blocos) {
                const d = b[b.type] || {};
                const rt = d.rich_text;
                const sub = b.has_children && !["child_page", "child_database", "synced_block"].includes(b.type) ? await filhos(b.id) : [];
                if (b.type !== "numbered_list_item") n = 0;
                switch (b.type) {
                  case "heading_1":
                  case "heading_2": {
                    const chave = SECOES[norm(texto(rt))];
                    if (chave && nivel === 0) { atual = chave; break; }
                    add("### " + inline(rt));
                    break;
                  }
                  case "heading_3": add("### " + inline(rt)); break;
                  case "paragraph": { const s = inline(rt); if (s) add(recuo + s); break; }
                  case "bulleted_list_item":
                  case "to_do": add(recuo + "- " + inline(rt)); break;
                  case "numbered_list_item": n++; add(recuo + n + ". " + inline(rt)); break;
                  case "quote": add("> " + inline(rt)); break;
                  case "callout": {
                    const icone = d.icon && d.icon.emoji;
                    const s = inline(rt);
                    const m = s.match(/(\d+)\s*foto/);
                    if (icone === "📷" || /fotos e v[ií]deos entram no portal/i.test(s)) { ficha.midias += m ? Number(m[1]) : 0; break; }
                    if (s) add("> " + s);
                    break;
                  }
                  case "toggle": add(recuo + "**" + inline(rt).replace(/\*\*/g, "") + "**"); break;
                  case "table_row": add(recuo + "- " + (d.cells || []).map(inline).filter(Boolean).join(" · ")); break;
                  case "image": case "video": case "file": case "pdf": case "embed": ficha.midias++; break;
                  default: break;
                }
                if (sub.length) await percorrer(sub, ["toggle", "table", "column_list", "column"].includes(b.type) ? nivel : nivel + 1);
              }
            }
            await percorrer(await filhos(pageId), 0);
            for (const k of Object.keys(ficha.secoes)) {
              const s = ficha.secoes[k].join("\n").replace(/\n{3,}/g, "\n\n").trim();
              if (s) ficha.secoes[k] = s; else delete ficha.secoes[k];
            }
            return ficha;
          }

          // ---------- principal ----------
          async function main() {
            if (!process.env.NOTION_TOKEN) throw new Falha("Falta o segredo NOTION_TOKEN no GitHub.");
            if (!process.env.ROBO_SENHA) throw new Falha("Falta o segredo ROBO_SENHA no GitHub.");
            if (!process.env.ROBO_EMAIL) throw new Falha("Falta o ROBO_EMAIL no arquivo do robô.");
            if (!DB_ID) throw new Falha("Falta o NOTION_DATABASE_ID no arquivo do robô.");
            const app = initializeApp(FIREBASE_CONFIG);
            try {
              await signInWithEmailAndPassword(getAuth(app), process.env.ROBO_EMAIL, process.env.ROBO_SENHA);
            } catch (e) {
              const c = (e && e.code) || "";
              if (/invalid-credential|wrong-password|user-not-found|invalid-email|invalid-login/.test(c)) throw new Falha("O Firebase recusou o login do robô. Confira se o usuário " + process.env.ROBO_EMAIL + " existe em Authentication e se a senha no segredo ROBO_SENHA é a mesma.");
              if (/too-many-requests/.test(c)) throw new Falha("O Firebase bloqueou temporariamente o login do robô por muitas tentativas. Tente de novo mais tarde.");
              throw new Falha("Não foi possível entrar no Firebase (" + (c || "erro desconhecido") + ").");
            }
            const db = getFirestore(app);

            // 1. fichas publicadas
            const paginas = [];
            let cursor;
            do {
              const consulta = { filter: { property: "Status", status: { equals: "Publicado" } }, page_size: 100, ...(cursor ? { start_cursor: cursor } : {}) };
              let r;
              try {
                r = await notion("databases/" + DB_ID + "/query", consulta);
              } catch (e) {
                // Bancos novos do Notion também respondem pelo endereço de "fonte de dados".
                if (!(e instanceof Falha) || e.status !== 404 || !DS_ID) throw e;
                r = await notion("data_sources/" + DS_ID + "/query", consulta, "2025-09-03");
              }
              paginas.push(...r.results.filter((p) => !p.archived && !p.in_trash));
              cursor = r.has_more ? r.next_cursor : null;
            } while (cursor);
            if (!paginas.length) throw new Falha("Nenhuma ficha com Status “Publicado” foi encontrada. Nada foi alterado no portal.");

            // 2. monta os documentos
            const destinos = {}, passeios = {};
            let semDestino = 0;
            for (const pg of paginas) {
              const P = pg.properties || {};
              const nomeDestino = prop(P["Destino"]);
              if (!DESTINOS[nomeDestino]) { semDestino++; continue; }
              const [destId, destOrdem] = DESTINOS[nomeDestino];
              const conteudo = await lerPagina(pg.id);
              const comum = {
                nome: prop(P["Passeio"]) || "Sem título",
                categoria: prop(P["Categoria"]) || "",
                subtitulo: prop(P["Subtítulo"]) || "",
                duracao: prop(P["Duração"]) || "",
                altitude: prop(P["Altitude"]) || "",
                grupo: prop(P["Grupo"]) || "",
                preco: prop(P["Preço"]) || "",
                esforco: prop(P["Esforço"]) || "",
                etiquetas: prop(P["Etiquetas"]) || [],
                revisadoPor: (prop(P["Revisado por"]) || []).join(", "),
                revisadoEm: prop(P["Revisado em"]) || "",
                editadoEm: pg.last_edited_time || "",
                secoes: conteudo.secoes,
                midias: conteudo.midias,
                notion: pg.url || "",
                fonte: "notion",
              };
              const tudo = Object.values(conteudo.secoes).join("\n\n");
              if (comum.categoria === "Visão geral do destino") {
                destinos[destId] = { ...comum, id: destId, nome: nomeDestino, titulo: comum.nome, ordem: destOrdem, resumo: tudo };
              } else {
                const ordem = typeof prop(P["Ordem"]) === "number" ? prop(P["Ordem"]) : 999;
                passeios[pg.id.replace(/-/g, "")] = { ...comum, destino: destId, ordem, conteudo: tudo };
                if (!destinos[destId]) destinos[destId] = { id: destId, nome: nomeDestino, ordem: destOrdem, resumo: "", secoes: {}, midias: 0, notion: "", fonte: "notion" };
              }
            }

            // 3. grava (substitui por completo) e remove o que saiu do Notion
            let antigosD, antigosP;
            try {
              [antigosD, antigosP] = await Promise.all([getDocs(collection(db, "destinos")), getDocs(collection(db, "passeios"))]);
            } catch (e) {
              if (e && e.code === "permission-denied") throw new Falha("O Firebase não deixou o robô mexer nos passeios. Confira se as regras novas do Firestore foram publicadas (com o e-mail do robô).");
              throw e;
            }
            const ops = [];
            for (const [id, d] of Object.entries(destinos)) ops.push(["set", doc(db, "destinos", id), d]);
            for (const [id, p] of Object.entries(passeios)) ops.push(["set", doc(db, "passeios", id), p]);
            let removidos = 0;
            antigosD.forEach((doc) => { if (!destinos[doc.id]) { ops.push(["delete", doc.ref]); removidos++; } });
            antigosP.forEach((doc) => { if (!passeios[doc.id]) { ops.push(["delete", doc.ref]); removidos++; } });
            for (let i = 0; i < ops.length; i += 400) {
              const lote = writeBatch(db);
              for (const [tipo, ref, dados] of ops.slice(i, i + 400)) tipo === "set" ? lote.set(ref, dados) : lote.delete(ref);
              await lote.commit();
            }

            console.log("Notion: " + paginas.length + " fichas publicadas lidas.");
            console.log("Portal: " + Object.keys(destinos).length + " destinos e " + Object.keys(passeios).length + " passeios gravados; " + removidos + " itens antigos removidos.");
            if (semDestino) console.log("Atenção: " + semDestino + " ficha(s) publicada(s) sem Destino preenchido ficaram de fora.");
            await terminate(db);
          }

          main().then(() => process.exit(0)).catch((e) => {
            if (e instanceof Falha) console.error("Não deu certo: " + e.message);
            else console.error("Não deu certo: erro inesperado. " + String((e && e.message) || e).slice(0, 300));
            process.exit(1);
          });
          FIM_DO_ROBO
          node sincronizar.cjs# Portal-Araya-Expedicoes
