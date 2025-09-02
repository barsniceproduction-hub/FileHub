<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>FileHub Demo</title>
<style>
  body{font-family:Arial,sans-serif;background:#0f1724;color:#eee;padding:20px;}
  h1{margin-bottom:10px;}
  input,button,select{padding:8px;margin:5px;border-radius:6px;border:none;}
  button{cursor:pointer;background:#3bd1ff;color:#012;font-weight:700;}
  .file-list{margin-top:20px}
  .file-item{margin-bottom:6px;}
  .small{color:#aaa;font-size:12px;margin-left:6px;}
</style>
</head>
<body>
<h1>FileHub Demo</h1>

<div id="adminArea">
  <h3>Загрузка файла (только для админа)</h3>
  <input type="file" id="fileInput">
  <button id="uploadBtn">Загрузить</button>
  <div id="uploadMsg"></div>
  <div>
    Сортировать: 
    <select id="sortSelect">
      <option value="name">По имени</option>
      <option value="date">По дате</option>
      <option value="size">По размеру</option>
    </select>
  </div>
</div>

<h3>Доступные файлы:</h3>
<div class="file-list" id="fileList"></div>

<script>
// Пароль админа
const ADMIN_PASS = "1234";

let db;
const request = indexedDB.open("filehubDB", 1);
request.onupgradeneeded = e=>{
  db = e.target.result;
  if(!db.objectStoreNames.contains("files")){
    const store = db.createObjectStore("files",{keyPath:"name"});
    store.createIndex("date","date",{unique:false});
    store.createIndex("size","size",{unique:false});
  }
};
request.onsuccess = e=>{db=e.target.result; renderFiles();}
request.onerror = e=>console.error(e);

const fileInput = document.getElementById("fileInput");
const uploadBtn = document.getElementById("uploadBtn");
const uploadMsg = document.getElementById("uploadMsg");
const fileList = document.getElementById("fileList");
const sortSelect = document.getElementById("sortSelect");

uploadBtn.onclick = ()=>{
  const file = fileInput.files[0];
  if(!file) return alert("Выберите файл");
  const pass = prompt("Введите пароль админа:");
  if(pass !== ADMIN_PASS) return alert("Неверный пароль!");
  
  const tx = db.transaction("files","readwrite");
  const store = tx.objectStore("files");
  store.put({name:file.name, blob:file, date:Date.now(), size:file.size});
  tx.oncomplete = ()=>{uploadMsg.textContent=`Файл "${file.name}" успешно загружен`; fileInput.value=""; renderFiles();}
  tx.onerror = ()=>uploadMsg.textContent="Ошибка при загрузке";
}

function renderFiles(){
  const tx = db.transaction("files","readonly");
  const store = tx.objectStore("files");
  const req = store.getAll();
  req.onsuccess = ()=>{
    let files=req.result;
    const sortBy = sortSelect.value;
    if(sortBy==="name") files.sort((a,b)=>a.name.localeCompare(b.name));
    if(sortBy==="date") files.sort((a,b)=>b.date - a.date);
    if(sortBy==="size") files.sort((a,b)=>b.size - a.size);

    fileList.innerHTML="";
    if(files.length===0){fileList.textContent="Файлов пока нет"; return;}
    files.forEach(f=>{
      const div = document.createElement("div");
      div.className="file-item";

      const downloadBtn = document.createElement("button");
      downloadBtn.textContent="Скачать";
      downloadBtn.onclick=()=>{
        const url = URL.createObjectURL(f.blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = f.name;
        a.click();
        URL.revokeObjectURL(url);
      }

      const deleteBtn = document.createElement("button");
      deleteBtn.textContent="Удалить";
      deleteBtn.onclick=()=>{
        const pass = prompt("Введите пароль админа для удаления:");
        if(pass !== ADMIN_PASS) return alert("Неверный пароль!");
        const tx = db.transaction("files","readwrite");
        const store = tx.objectStore("files");
        store.delete(f.name);
        tx.oncomplete = renderFiles;
      }

      const sizeKB = (f.size/1024).toFixed(1);
      const dateStr = new Date(f.date).toLocaleString();
      div.innerHTML=`<strong>${f.name}</strong> <span class="small">[${sizeKB} KB, ${dateStr}]</span> `;
      div.appendChild(downloadBtn);
      div.appendChild(deleteBtn);
      fileList.appendChild(div);
    });
  }
}

sortSelect.onchange=renderFiles;
</script>
</body>
</html>
