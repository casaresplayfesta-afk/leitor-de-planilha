<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>CSV por Categoria</title>
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>
<style>
body { font-family: Arial, sans-serif; background:#f5f5f5; padding:20px; }
h1 { text-align:center; }
input, button { padding:8px; margin:5px; border-radius:6px; border:1px solid #ccc; }
#gavetas { display:flex; gap:10px; flex-wrap:wrap; margin-bottom:20px; }
.gaveta { background:#4285F4; color:white; padding:15px; border-radius:8px; flex:1 1 150px; text-align:center; box-shadow:0 2px 5px rgba(0,0,0,0.2); cursor:pointer; transition: transform 0.2s;}
.gaveta:hover { transform: scale(1.05); }
.gaveta h3 { margin:0 0 5px 0; font-size:18px; }
.gaveta p { font-size:22px; margin:0; font-weight:bold; }
table { width:100%; border-collapse:collapse; margin-top:10px; background:#fff; }
th, td { border:1px solid #ccc; padding:8px; text-align:left; }
th { background:#333; color:white; }
tr:nth-child(even) { background-color:#f2f2f2; }
.filtro { margin-top:5px; margin-bottom:10px; padding:6px; width:200px; }
h2 { margin-top:20px; }
</style>
</head>
<body>

<h1>📊 CSV por Categoria</h1>

<input type="file" id="csvFile" accept=".csv">
<button onclick="baixarExcel()">Baixar Excel por Categoria</button>

<div id="gavetas"></div>
<div id="tabelas"></div>

<script>
let linhas = [];
let cabecalho = [];
let categorias = [];
let categoriaIndex = -1;
let nomeIndex = -1;

document.getElementById("csvFile").addEventListener("change", function(e){
    const file = e.target.files[0];
    if(!file) return;

    Papa.parse(file, {
        header: true,
        skipEmptyLines: true,
        complete: function(results) {
            linhas = results.data;
            cabecalho = results.meta.fields;

            categoriaIndex = cabecalho.findIndex(h => h.trim().toLowerCase() === "categoria");
            if(categoriaIndex === -1){
                alert("Coluna 'Categoria' não encontrada!");
                return;
            }

            nomeIndex = cabecalho.findIndex(h => h.trim().toLowerCase() === "nome");
            if(nomeIndex === -1){
                alert("Coluna 'Nome' não encontrada! As linhas não serão ordenadas.");
            }

            // Criar lista única de categorias separando múltiplas por vírgula
            categorias = [...new Set(
                linhas.flatMap(l => l[cabecalho[categoriaIndex]]
                    .split(',')
                    .map(c => c.trim())
                )
            )].sort((a,b) => a.localeCompare(b));

            atualizarGavetas();
            mostrarTabelasPorCategoria();
        }
    });
});

function atualizarGavetas(){
    const div = document.getElementById("gavetas");
    div.innerHTML = "";
    categorias.forEach(cat=>{
        const linhasCat = linhas.filter(l => 
            l[cabecalho[categoriaIndex]].split(',').map(c=>c.trim()).includes(cat)
        );
        const count = linhasCat.length;
        const gaveta = document.createElement("div");
        gaveta.className = "gaveta";
        gaveta.innerHTML = `<h3>${cat}</h3><p>${count}</p>`;
        gaveta.onclick = () => {
            const elemento = document.getElementById("categoria-"+cat.replace(/\s+/g,"_"));
            if(elemento) elemento.scrollIntoView({behavior: "smooth", block: "start"});
        };
        div.appendChild(gaveta);
    });
}

function mostrarTabelasPorCategoria(){
    const div = document.getElementById("tabelas");
    div.innerHTML = "";

    categorias.forEach(cat=>{
        let linhasCat = linhas.filter(l => 
            l[cabecalho[categoriaIndex]].split(',').map(c=>c.trim()).includes(cat)
        );

        if(nomeIndex !== -1){
            linhasCat.sort((a,b) => (a[cabecalho[nomeIndex]]||"").localeCompare(b[cabecalho[nomeIndex]]||""));
        }

        if(linhasCat.length){
            let html = `<div id="categoria-${cat.replace(/\s+/g,"_")}"><h2>Categoria: ${cat}</h2>`;
            html += `<input type="text" class="filtro" placeholder="Buscar na categoria ${cat}" oninput="filtrarTabela(this, '${cat}')">`;
            html += "<table><thead><tr>"+cabecalho.map(c=>`<th>${c}</th>`).join("")+"</tr></thead><tbody>";
            linhasCat.forEach(l=>{
                html += "<tr>"+cabecalho.map(col => `<td>${l[col]}</td>`).join("")+"</tr>";
            });
            html += "</tbody></table></div>";
            div.innerHTML += html;
        }
    });
}

function filtrarTabela(input, cat){
    const tabela = input.nextElementSibling;
    const trs = tabela.querySelectorAll("tbody tr");
    trs.forEach(tr=>{
        const mostrar = Array.from(tr.cells).some(td=>td.textContent.toLowerCase().includes(input.value.toLowerCase()));
        tr.style.display = mostrar ? "" : "none";
    });
}

function baixarExcel(){
    if(!linhas.length){ alert("Faça upload do CSV primeiro!"); return; }

    const wb = XLSX.utils.book_new();

    categorias.forEach(cat=>{
        let linhasCat = linhas.filter(l => 
            l[cabecalho[categoriaIndex]].split(',').map(c=>c.trim()).includes(cat)
        );

        if(nomeIndex !== -1){
            linhasCat.sort((a,b) => (a[cabecalho[nomeIndex]]||"").localeCompare(b[cabecalho[nomeIndex]]||""));
        }

        if(linhasCat.length){
            const wsData = [cabecalho, ...linhasCat.map(l => cabecalho.map(col => (l[col] || "").toString()))];
            const ws = XLSX.utils.aoa_to_sheet(wsData);
            const aba = cat.replace(/[:\/\\?*\[\]]/g,'_').substring(0,31);
            XLSX.utils.book_append_sheet(wb, ws, aba);
        }
    });

    XLSX.writeFile(wb, "PlanilhaPorCategoria.xlsx");
}
</script>

</body>
</html>
