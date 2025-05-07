✅ 1. Configuração Inicial

node -v       # Verifique o Node
npm -v        # Verifique o NPM

mkdir api-tarefas
cd api-tarefas
npm init -y   # Inicialize o projeto

✅ 2. Instalação dos Pacotes
npm install express uuid joi
npm install morgan --save-dev  # Opcional

✅ 3. Estrutura do Projeto

mkdir -p src/{controllers,routes,services,middlewares,utils,database}
touch src/{app.js}
touch src/controllers/tarefasController.js
touch src/routes/tarefasRoutes.js
touch src/services/tarefasService.js
touch src/middlewares/validateTarefa.js
touch src/utils/logger.js
touch src/database/fakeDb.js

✅ 4. Implementação por Módulo

// src/database/fakeDb.js
let tarefas = [];

module.exports = {
  getTarefas: () => tarefas,
  setTarefas: (novaLista) => { tarefas = novaLista; }
};

🔹 logger.js (Logs do sistema)
// src/utils/logger.js
module.exports = {
  log: (mensagem) => {
    console.log(`[${new Date().toISOString()}] ${mensagem}`);
  }
};

🔹 validateTarefa.js (Validação de dados com Joi)
// src/middlewares/validateTarefa.js
const Joi = require('joi');

const tarefaSchema = Joi.object({
  titulo: Joi.string().min(3).required(),
  descricao: Joi.string().required(),
  concluida: Joi.boolean().required()
});

module.exports = (req, res, next) => {
  const { error } = tarefaSchema.validate(req.body);
  if (error) {
    return res.status(400).json({ erro: error.details[0].message });
  }
  next();
};

🔹 tarefasService.js (Camada de serviço)
// src/services/tarefasService.js
const { v4: uuidv4 } = require('uuid');
const db = require('../database/fakeDb');

function listarTarefas(filtroConcluida) {
  const tarefas = db.getTarefas();
  if (filtroConcluida !== undefined) {
    return tarefas.filter(t => String(t.concluida) === filtroConcluida);
  }
  return tarefas;
}

function buscarPorId(id) {
  return db.getTarefas().find(t => t.id === id);
}

function adicionarTarefa(tarefa) {
  const nova = { id: uuidv4(), ...tarefa };
  const tarefas = db.getTarefas();
  tarefas.push(nova);
  db.setTarefas(tarefas);
  return nova;
}

function atualizarTarefa(id, dados) {
  const tarefas = db.getTarefas();
  const index = tarefas.findIndex(t => t.id === id);
  if (index === -1) return null;
  tarefas[index] = { id, ...dados };
  db.setTarefas(tarefas);
  return tarefas[index];
}

function marcarComoConcluida(id) {
  const tarefas = db.getTarefas();
  const tarefa = tarefas.find(t => t.id === id);
  if (!tarefa) return null;
  tarefa.concluida = true;
  return tarefa;
}

function deletarTarefa(id) {
  const tarefas = db.getTarefas();
  const novaLista = tarefas.filter(t => t.id !== id);
  if (novaLista.length === tarefas.length) return false;
  db.setTarefas(novaLista);
  return true;
}

module.exports = {
  listarTarefas,
  buscarPorId,
  adicionarTarefa,
  atualizarTarefa,
  marcarComoConcluida,
  deletarTarefa
};
🔹 tarefasController.js (Orquestra requisições/respostas)

// src/controllers/tarefasController.js
const service = require('../services/tarefasService');
const logger = require('../utils/logger');

exports.listar = (req, res) => {
  const { concluida } = req.query;
  const tarefas = service.listarTarefas(concluida);
  res.json(tarefas);
};

exports.buscar = (req, res) => {
  const tarefa = service.buscarPorId(req.params.id);
  if (!tarefa) return res.status(404).json({ erro: 'Tarefa não encontrada' });
  res.json(tarefa);
};

exports.criar = (req, res) => {
  const nova = service.adicionarTarefa(req.body);
  logger.log('Tarefa criada com sucesso.');
  res.status(201).json(nova);
};

exports.atualizar = (req, res) => {
  const atualizada = service.atualizarTarefa(req.params.id, req.body);
  if (!atualizada) return res.status(404).json({ erro: 'Tarefa não encontrada' });
  res.json(atualizada);
};

exports.concluir = (req, res) => {
  const tarefa = service.marcarComoConcluida(req.params.id);
  if (!tarefa) return res.status(404).json({ erro: 'Tarefa não encontrada' });
  res.json({ mensagem: 'Tarefa marcada como concluída.' });
};

exports.deletar = (req, res) => {
  const sucesso = service.deletarTarefa(req.params.id);
  if (!sucesso) return res.status(404).json({ erro: 'Tarefa não encontrada' });
  logger.log('Tarefa deletada.');
  res.status(204).send();
};

🔹 tarefasRoutes.js (Rotas RESTful)
// src/routes/tarefasRoutes.js
const express = require('express');
const router = express.Router();
const controller = require('../controllers/tarefasController');
const validate = require('../middlewares/validateTarefa');

router.get('/', controller.listar);
router.get('/:id', controller.buscar);
router.post('/', validate, controller.criar);
router.put('/:id', validate, controller.atualizar);
router.patch('/:id/concluir', controller.concluir);
router.delete('/:id', controller.deletar);

module.exports = router;

🔹 app.js (Configura e inicia o app)

// src/app.js
const express = require('express');
const morgan = require('morgan');
const tarefasRoutes = require('./routes/tarefasRoutes');

const app = express();
app.use(express.json());
app.use(morgan('dev')); // log básico

app.use('/tarefas', tarefasRoutes);

app.use((req, res) => {
  res.status(404).json({ erro: 'Rota não encontrada' });
});

const PORT = 3000;
app.listen(PORT, () => {
  console.log(`Servidor rodando na porta ${PORT}`);
});





