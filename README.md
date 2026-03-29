<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>Analisador de Balancete - BMD</title>

<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>

<style>
body {
  font-family: Arial;
  background: #111;
  color: white;
  text-align: center;
  padding: 20px;
}

/* 🔥 LOGO */
.logo {
  font-size: 50px;
  font-weight: bold;
  letter-spacing: 5px;
  background: linear-gradient(45deg, #00ffcc, #00aa88);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  margin-bottom: 10px;
}

h1 { color: #00ffcc; }

/* ABAS */
.tabs {
  display: flex;
  justify-content: center;
  margin-bottom: 20px;
}

.tab {
  padding: 10px 20px;
  cursor: pointer;
  background: #333;
  margin: 5px;
  border-radius: 5px;
}

.tab.active {
  background: #00aa88;
}

.conteudo { display: none; }
.conteudo.active { display: block; }

/* INPUT */
input { margin: 20px; padding: 10px; }

/* BOTÕES */
button {
  padding: 10px 20px;
  margin: 10px;
  background: #00aa88;
  border: none;
  border-radius: 5px;
  color: white;
  cursor: pointer;
}

/* COLUNAS */
.box {
  display: flex;
  justify-content: space-around;
  margin-top: 20px;
}

.coluna {
  width: 45%;
  padding: 15px;
  border-radius: 10px;
}

.positivo { background: #063; }
.negativo { background: #600; }

/* CONTADOR */
.contador {
  font-size: 18px;
  margin-bottom: 10px;
  font-weight: bold;
}

li { margin: 8px 0; text-align: left; }
</style>
</head>

<body>

<div class="logo">BMD</div>
<h1>📄 Analisador de Balancete</h1>

<div class="tabs">
  <div class="tab active" onclick="trocarAba(1)">Sistema 1</div>
  <div class="tab" onclick="trocarAba(2)">Sistema 2</div>
</div>

<!-- ABA 1 -->
<div id="aba1" class="conteudo active">
  <input type="file" id="pdfInput1" multiple>
  <br>
  <button onclick="baixarZIP(1, 'negativos')">⬇️ Negativos</button>
  <button onclick="baixarZIP(1, 'positivos')">⬇️ Positivos</button>

  <div class="box">
    <div class="coluna positivo">
      <div class="contador">Positivos: <span id="countPos1">0</span></div>
      <ul id="positivos1"></ul>
    </div>

    <div class="coluna negativo">
      <div class="contador">Negativos: <span id="countNeg1">0</span></div>
      <ul id="negativos1"></ul>
    </div>
  </div>
</div>

<!-- ABA 2 -->
<div id="aba2" class="conteudo">
  <input type="file" id="pdfInput2" multiple>
  <br>
  <button onclick="baixarZIP(2, 'negativos')">⬇️ Negativos</button>
  <button onclick="baixarZIP(2, 'positivos')">⬇️ Positivos</button>

  <div class="box">
    <div class="coluna positivo">
      <div class="contador">Positivos: <span id="countPos2">0</span></div>
      <ul id="positivos2"></ul>
    </div>

    <div class="coluna negativo">
      <div class="contador">Negativos: <span id="countNeg2">0</span></div>
      <ul id="negativos2"></ul>
    </div>
  </div>
</div>

<script>

// ABAS
function trocarAba(num) {
  document.querySelectorAll(".tab").forEach(t => t.classList.remove("active"));
  document.querySelectorAll(".conteudo").forEach(c => c.classList.remove("active"));

  document.querySelectorAll(".tab")[num-1].classList.add("active");
  document.getElementById("aba" + num).classList.add("active");
}

// DADOS
const dados = {
  1: { negativos: [], positivos: [] },
  2: { negativos: [], positivos: [] }
};

// INPUTS
document.getElementById("pdfInput1").addEventListener("change", (e)=>processar(e,1));
document.getElementById("pdfInput2").addEventListener("change", (e)=>processar(e,2));

// PROCESSAR
async function processar(event, aba) {

  document.getElementById("positivos"+aba).innerHTML = "";
  document.getElementById("negativos"+aba).innerHTML = "";

  dados[aba].negativos = [];
  dados[aba].positivos = [];

  atualizarContador(aba);

  for (let file of event.target.files) {
    const texto = await lerPDF(file);
    analisar(texto, file.name, file, aba);
  }
}

// LER PDF
async function lerPDF(file) {
  const reader = new FileReader();

  return new Promise(resolve => {
    reader.onload = async function () {
      const pdf = await pdfjsLib.getDocument(new Uint8Array(this.result)).promise;
      let texto = "";

      for (let i = 1; i <= pdf.numPages; i++) {
        const pagina = await pdf.getPage(i);
        const conteudo = await pagina.getTextContent();
        conteudo.items.forEach(item => texto += item.str + " ");
      }

      resolve(texto.toLowerCase());
    };

    reader.readAsArrayBuffer(file);
  });
}

// ANALISAR
function analisar(texto, nome, file, aba) {

  texto = texto.replace(/\s+/g, " ");
  const index = texto.indexOf("resultado do período");

  if (index !== -1) {

    const trecho = texto.substring(index, 300 + index);
    const valores = trecho.match(/\(?\s*\d{1,3}(?:\.\d{3})*,\d{2}\s*\)?/g);

    if (valores && valores.length >= 4) {

      const saldo = valores[3].replace(/\s+/g, "");
      const ehNegativo = saldo.includes("(");

      if (ehNegativo) {
        adicionar("negativos"+aba, nome, saldo);
        dados[aba].negativos.push(file);
      } else {
        adicionar("positivos"+aba, nome, saldo);
        dados[aba].positivos.push(file);
      }

      atualizarContador(aba);
    }
  }
}

// ADICIONAR LISTA
function adicionar(lista, nome, saldo) {
  const li = document.createElement("li");
  li.textContent = `${nome} → ${saldo}`;
  document.getElementById(lista).appendChild(li);
}

// CONTADOR
function atualizarContador(aba) {
  document.getElementById("countPos"+aba).textContent = dados[aba].positivos.length;
  document.getElementById("countNeg"+aba).textContent = dados[aba].negativos.length;
}

// DOWNLOAD ZIP
async function baixarZIP(aba, tipo) {

  if (dados[aba][tipo].length === 0) {
    alert("Nenhum arquivo encontrado.");
    return;
  }

  const zip = new JSZip();

  dados[aba][tipo].forEach(f => zip.file(f.name, f));

  const blob = await zip.generateAsync({ type: "blob" });

  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = `${tipo}_aba${aba}.zip`;
  a.click();
}

</script>

</body>
</html>
