<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<title>BMD - Sistema Completo</title>

<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>

<style>
body {
  font-family: Arial;
  background: #111;
  color: white;
  text-align: center;
  padding: 20px;
}

.logo {
  font-size: 50px;
  font-weight: bold;
  letter-spacing: 5px;
  background: linear-gradient(45deg, #00ffcc, #00aa88);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.tabs { display: flex; justify-content: center; margin: 20px; }
.tab { padding: 10px 20px; background: #333; margin: 5px; cursor: pointer; border-radius: 5px; }
.tab.active { background: #00aa88; }

.conteudo { display: none; }
.conteudo.active { display: block; }

button {
  padding: 10px 20px;
  margin: 10px;
  background: #00aa88;
  border: none;
  color: white;
  cursor: pointer;
}

.box { display: flex; justify-content: space-around; }
.coluna { width: 45%; padding: 10px; border-radius: 10px; }
.positivo { background: #063; }
.negativo { background: #600; }

table { width: 95%; margin: 20px auto; border-collapse: collapse; }
th, td { border: 1px solid #fff; padding: 8px; }
th { background: #00aa88; }
td { background: #063; }

.maior { background: #0044ff; }
.menor { background: #880000; }
</style>
</head>

<body>

<div class="logo">BMD</div>

<div class="tabs">
  <div class="tab active" onclick="trocarAba(1)">Sistema 1</div>
  <div class="tab" onclick="trocarAba(2)">Sistema 2</div>
</div>

<!-- 🔥 SISTEMA 1 -->
<div id="aba1" class="conteudo active">

<input type="file" id="pdf1" multiple><br>
<button onclick="baixarZIP('negativos')">Negativos ZIP</button>
<button onclick="baixarZIP('positivos')">Positivos ZIP</button>

<div class="box">
<div class="coluna positivo">
<h3>Positivos (<span id="cPos">0</span>)</h3>
<ul id="pos"></ul>
</div>

<div class="coluna negativo">
<h3>Negativos (<span id="cNeg">0</span>)</h3>
<ul id="neg"></ul>
</div>
</div>

</div>

<!-- 🔥 SISTEMA 2 -->
<div id="aba2" class="conteudo">

<input type="file" id="pdf2" multiple><br>
<button onclick="exportarExcel()">Exportar Excel</button>

<table id="tabela">
<thead>
<tr>
<th>Arquivo</th>
<th>Empresa</th>
<th>Resultado</th>
<th>Produtos</th>
<th>Mercadoria</th>
<th>Serviços</th>
<th>Simples</th>
<th>Total Serviços</th>
<th>Comp</th>
<th>Total Geral</th>
<th>Comp</th>
</tr>
</thead>
<tbody id="tbody"></tbody>
</table>

</div>

<script>

// 🔥 ABAS
function trocarAba(n){
document.querySelectorAll(".tab").forEach(t=>t.classList.remove("active"));
document.querySelectorAll(".conteudo").forEach(c=>c.classList.remove("active"));
document.querySelectorAll(".tab")[n-1].classList.add("active");
document.getElementById("aba"+n).classList.add("active");
}

// 🔥 SISTEMA 1
let posFiles=[], negFiles=[];

document.getElementById("pdf1").addEventListener("change", async e=>{
pos.innerHTML=""; neg.innerHTML="";
posFiles=[]; negFiles=[];

for(let f of e.target.files){
let t=await lerPDF(f);
let trecho=t.substring(t.indexOf("resultado do período"),300);
let val=trecho.match(/\(?\d{1,3}(?:\.\d{3})*,\d{2}\)?/g);

if(val && val[3]){
let saldo=val[3];
if(saldo.includes("(")){
neg.innerHTML+=`<li>${f.name} → ${saldo}</li>`;
negFiles.push(f);
}else{
pos.innerHTML+=`<li>${f.name} → ${saldo}</li>`;
posFiles.push(f);
}
}
}
cPos.textContent=posFiles.length;
cNeg.textContent=negFiles.length;
});

// ZIP
async function baixarZIP(tipo){
let lista= tipo=="negativos"?negFiles:posFiles;
if(!lista.length)return alert("Nenhum arquivo");

let zip=new JSZip();
lista.forEach(f=>zip.file(f.name,f));
let blob=await zip.generateAsync({type:"blob"});

let a=document.createElement("a");
a.href=URL.createObjectURL(blob);
a.download=tipo+".zip";
a.click();
}

// 🔥 SISTEMA 2
document.getElementById("pdf2").addEventListener("change", async e=>{
tbody.innerHTML="";
for(let f of e.target.files){
let t=await lerPDF(f);
processarResumo(t,f.name);
}
});

// 🏢 PEGAR NOME EMPRESA
function pegarNomeEmpresa(texto){
const linhas = texto.split("\n");

for(let linha of linhas){
linha = linha.trim();

if(
linha.length > 5 &&
!linha.includes("página") &&
!linha.includes("balancete") &&
!linha.includes("contábil")
){
return linha.toUpperCase();
}
}
return "NÃO IDENTIFICADO";
}

// FUNÇÕES AUX
function pegarValor(linha){
if(!linha)return "-";
let n=linha.match(/\(?\d{1,3}(?:\.\d{3})*,\d{2}\)?/g);
return n?n[n.length-1]:"-";
}

function buscar(texto,cod){
return texto.split("\n").find(l=>l.startsWith(cod+" "))||"";
}

function num(v){
if(!v || v=="-") return 0;
return parseFloat(v.replace(/\./g,"").replace(",",".").replace("(","-").replace(")",""));
}

// 🔥 PROCESSAR RESUMO
function processarResumo(t,nome){

let empresa = pegarNomeEmpresa(t);

let r=pegarValor(buscar(t,"2600"));
let p=pegarValor(buscar(t,"2603"));
let m=pegarValor(buscar(t,"2652"));
let s=pegarValor(buscar(t,"2700"));
let si=pegarValor(buscar(t,"2831"));

let vr=num(r), vp=num(p), vm=num(m), vs=num(s), vsi=num(si);

let totalS=(vs*0.32)+(vsi*0.05);
let totalG=(vp*0.08)+(vm*0.08)+(vsi*0.05);

let c1= totalS>vr?"MAIOR":"MENOR";
let c2= totalG>vr?"MAIOR":"MENOR";

tbody.innerHTML+=`
<tr>
<td>${nome}</td>
<td>${empresa}</td>
<td>${r}</td>
<td>${p}</td>
<td>${m}</td>
<td>${s}</td>
<td>${si}</td>
<td>${totalS.toFixed(2)}</td>
<td class="${c1=='MAIOR'?'maior':'menor'}">${c1}</td>
<td>${totalG.toFixed(2)}</td>
<td class="${c2=='MAIOR'?'maior':'menor'}">${c2}</td>
</tr>`;
}

// EXCEL
function exportarExcel(){
let wb=XLSX.utils.table_to_book(document.getElementById("tabela"));
XLSX.writeFile(wb,"Resumo.xlsx");
}

// 🔥 LEITOR PDF
async function lerPDF(file){
const reader=new FileReader();
return new Promise(resolve=>{
reader.onload=async function(){
const pdf=await pdfjsLib.getDocument(new Uint8Array(this.result)).promise;
let txt="";
for(let i=1;i<=pdf.numPages;i++){
let p=await pdf.getPage(i);
let c=await p.getTextContent();
c.items.forEach(it=>txt+=it.str+" ");
txt+="\n";
}
resolve(txt.toLowerCase());
};
reader.readAsArrayBuffer(file);
});
}

</script>

</body>
</html>

</script>

</body>
</html>
