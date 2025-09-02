// server.js
const express = require("express");
const multer = require("multer");
const fs = require("fs");
const path = require("path");

const app = express();
const PORT = process.env.PORT || 3000;

// Папка для файлов
const UPLOAD_DIR = path.join(__dirname, "uploads");
if (!fs.existsSync(UPLOAD_DIR)) fs.mkdirSync(UPLOAD_DIR);

// Настройка Multer
const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, UPLOAD_DIR),
  filename: (req, file, cb) => cb(null, file.originalname)
});
const upload = multer({ storage });

// Админский пароль
const ADMIN_PASS = "1234"; // поменяй на свой

// Фронтенд
app.get("/", (req, res) => {
  res.send(`
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>FileHub</title>
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
<h1>FileHub</h1>
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
const fileInput = document.getElementById("fileInput");
const uploadBtn = document.getElementById("uploadBtn");
const uploadMsg = document.getElementById("uploadMsg");
const fileList = document.getElementById("fileList");
const sortSelect = document.getElementById("sortSelect");

async function loadFiles(){
  const res = await fetch("/files");
  let files = await res.json();
  const sortBy = sortSelect.value;
  if(sortBy==="name") files.sort((a,b)=>a.name.localeCompare(b.name));
  if(sortBy==="date") files.sort((a,b)=>b.date - a.date);
  if(sortBy==="size") files.sort((a,b)=>b.size - a.size);

  fileList.innerHTML = "";
  if(files.length===0){fileList.textContent="Файлов пока нет"; return;}
  files.forEach(f=>{
    const div = document.createElement("div");
    div.className = "file-item";
    const downloadBtn = document.createElement("button");
    downloadBtn.textContent = "Скачать";
    downloadBtn.onclick = ()=>window.open("/download/"+encodeURIComponent(f.name),"__blank");

    const deleteBtn = document.createElement("button");
    deleteBtn.textContent = "Удалить";
    deleteBtn.onclick = async ()=>{
      const pass = prompt("Введите пароль админа для удаления:");
      if(pass !== "${ADMIN_PASS}") return alert("Неверный пароль!");
      await fetch("/delete/"+encodeURIComponent(f.name),{method:"POST"});
      loadFiles();
    };

    const sizeKB = (f.size/1024).toFixed(1);
    const dateStr = new Date(f.date).toLocaleString();
    div.innerHTML = "<strong>"+f.name+"</strong> <span class='small'>["+sizeKB+" KB, "+dateStr+"]</span> ";
    div.appendChild(downloadBtn);
    div.appendChild(deleteBtn);
    fileList.appendChild(div);
  });
}

sortSelect.onchange = loadFiles;

// Загрузка файла
uploadBtn.onclick = async () => {
  const file = fileInput.files[0];
  if(!file) return alert("Выберите файл");
  const pass = prompt("Введите пароль админа:");
  if(pass !== "${ADMIN_PASS}") return alert("Неверный пароль!");

  const formData = new FormData();
  formData.append("file", file);

  const res = await fetch("/upload",{method:"POST",body:formData});
  const json = await res.json();
  if(json.success){
    uploadMsg.textContent = "Файл '" + file.name + "' успешно загружен";
    fileInput.value = "";
    loadFiles();
  } else uploadMsg.textContent="Ошибка при загрузке";
};

loadFiles();
</script>
</body>
</html>
  `);
});

// Загрузка файла
app.post("/upload", upload.single("file"), (req,res)=>{
  if(!req.file) return res.json({success:false});
  res.json({success:true});
});

// Список файлов
app.get("/files", (req,res)=>{
  fs.readdir(UPLOAD_DIR,(err,files)=>{
    if(err) return res.json([]);
    const result = files.map(f=>{
      const stat = fs.statSync(path.join(UPLOAD_DIR,f));
      return {name:f, size:stat.size, date:stat.mtimeMs};
    });
    res.json(result);
  });
});

// Скачивание
app.get("/download/:filename",(req,res)=>{
  const file = path.join(UPLOAD_DIR,req.params.filename);
  if(fs.existsSync(file)) res.download(file);
  else res.status(404).send("File not found");
});

// Удаление файла
app.post("/delete/:filename",(req,res)=>{
  const file = path.join(UPLOAD_DIR,req.params.filename);
  if(fs.existsSync(file)) fs.unlinkSync(file);
  res.json({success:true});
});

app.listen(PORT,()=>console.log("Server running on port "+PORT));
