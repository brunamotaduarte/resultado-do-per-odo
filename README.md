<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>Resumo PDF Positivos</title>

<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>

<style>
body { font-family: Arial; background: #111; color: white; text-align: center; padding: 20px; }
h1 { color: #00ffcc; }
table { width: 95%; margin: 20px auto; border-collapse: collapse; }
th, td { border: 1px solid #fff; padding: 8px; }
th { background: #00aa88; }
td { background: #063; }
.maior { background: blue; }
.menor { background: red; }
</style>
</head>

<body>

<h1>📊 Análise dos PDFs Positivos</h1>
<button onclick="exportarExcel()">📥 Exportar Excel</button>

<table id="tabela">
<thead>
<tr>
<th>Arquivo</th>
<th>Empresa</th>
<th>Resultado</th>
<th>Serviços+Simples</th>
<th>Comparação</th>
<th>Total Geral</th>
<th>Comparação</th>
</tr>
</thead>
<tbody id="tabelaResumo"></tbody>
</table>

<script>

pdfjsLib.GlobalWorkerOptions.workerSrc =
"https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.worker.min.js";

const dados = JSON.parse(localStorage.getItem("pdfsPositivos")) || [];

function base64ToFile(base64, nome) {
  const byteString = atob(base64.split(',')[1]);
  const ab = new ArrayBuffer(byteString.length);
  const ia = new Uint8Array(ab);

  for (let i = 0; i < byteString.length; i++) {
    ia[i] = byteString.charCodeAt(i);
  }

  return new File([ia], nome, { type: "application/pdf" });
}

for (let item of dados) {
  processarArquivo(item);
}

async function processarArquivo(item) {
  const file = base64ToFile(item.dados, item.nome);
  const texto = await lerPDF(file);
  processar(texto, item.nome);
}

async function lerPDF(file) {
  const reader = new FileReader();

  return new Promise((resolve) => {
    reader.onload = async function () {
      const pdf = await pdfjsLib.getDocument(new Uint8Array(this.result)).promise;

      let texto = "";

      for (let i = 1; i <= pdf.numPages; i++) {
        const pagina = await pdf.getPage(i);
        const conteudo = await pagina.getTextContent();
        conteudo.items.forEach(i => texto += i.str + " ");
        texto += "\n";
      }

      resolve(texto.toLowerCase());
    };

    reader.readAsArrayBuffer(file);
  });
}

// pegar empresa (primeira linha válida)
function pegarEmpresa(texto) {
  const linhas = texto.split("\n");
  for (let l of linhas) {
    l = l.trim();
    if (l.length > 5 && !l.includes("balancete")) {
      return l.toUpperCase();
    }
  }
  return "-";
}

function pegar(linha) {
  const nums = linha.match(/\(?\d{1,3}(?:\.\d{3})*,\d{2}\)?/g);
  return nums ? nums[nums.length - 1] : "-";
}

function num(v) {
  return parseFloat(v.replace(/\./g,"").replace(",",".").replace(/[()]/g,"")) || 0;
}

function processar(texto, nome) {

  const empresa = pegarEmpresa(texto);

  const resultado = pegar((texto.match(/2600.*?\n/)||[""])[0]);
  const serv = pegar((texto.match(/2700.*?\n/)||[""])[0]);
  const simp = pegar((texto.match(/2831.*?\n/)||[""])[0]);
  const prod = pegar((texto.match(/2603.*?\n/)||[""])[0]);
  const merc = pegar((texto.match(/2652.*?\n/)||[""])[0]);

  const total1 = num(serv)*0.32 + num(simp)*0.05;
  const total2 = num(prod)*0.08 + num(merc)*0.08 + num(simp)*0.05;

  const comp1 = total1 > num(resultado) ? "MAIOR" : "MENOR";
  const comp2 = total2 > num(resultado) ? "MAIOR" : "MENOR";

  const tr = document.createElement("tr");

  tr.innerHTML = `
  <td>${nome}</td>
  <td>${empresa}</td>
  <td>${resultado}</td>
  <td>${total1.toFixed(2)}</td>
  <td class="${comp1==="MAIOR"?"maior":"menor"}">${comp1}</td>
  <td>${total2.toFixed(2)}</td>
  <td class="${comp2==="MAIOR"?"maior":"menor"}">${comp2}</td>
  `;

  document.getElementById("tabelaResumo").appendChild(tr);
}

function exportarExcel(){
  const wb = XLSX.utils.table_to_book(document.getElementById("tabela"));
  XLSX.writeFile(wb,"resultado.xlsx");
}

</script>

</body>
</html>
