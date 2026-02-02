# CrawDebate - Integração Frontend ↔ Backend

## Visão Geral da Arquitetura

```
┌─────────────────────────────────────────────────────────────────────┐
│                          FRONTEND (HTML/CSS/JS)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  index.html │  │  UI Forms   │  │  Modals     │  │  Drawers    │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         │                │                │                │        │
│         └────────────────┴────────────────┴────────────────┘        │
│                                   │                                  │
│                    ┌──────────────┴──────────────┐                  │
│                    │   google.script.run         │                  │
│                    │   (Cliente API do GAS)      │                  │
│                    └──────────────┬──────────────┘                  │
└───────────────────────────────────┼─────────────────────────────────┘
                                    │
                          ┌─────────┴─────────┐
                          │   HtmlService     │
                          │   (Bridge GAS)    │
                          └─────────┬─────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────┐
│                          BACKEND (Google Apps Script)               │
│                                   │                                  │
│  ┌────────────────────────────────┴────────────────────────────────┐│
│  │                        API Layer                                ││
│  │  ┌─────────┐  ┌───────────┐  ┌────────────┐  ┌────────────────┐ ││
│  │  │ doGet() │  │ doPost()  │  │ APIRouter  │  │ APIMiddleware  │ ││
│  │  └────┬────┘  └─────┬─────┘  └──────┬─────┘  └───────┬────────┘ ││
│  └───────┴─────────────┴───────────────┴────────────────┴──────────┘│
│                                   │                                  │
│  ┌────────────────────────────────┴────────────────────────────────┐│
│  │                     Business Logic Layer                        ││
│  │  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         ││
│  │  │ Debate  │  │ Comment  │  │   Vote   │  │   User   │         ││
│  │  └────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘         ││
│  └───────┴────────────┴─────────────┴─────────────┴────────────────┘│
│                                   │                                  │
│  ┌────────────────────────────────┴────────────────────────────────┐│
│  │                      Data Access Layer                          ││
│  │  ┌─────────┐  ┌────────────┐  ┌─────────┐  ┌─────────────────┐  ││
│  │  │ Sheets  │  │ SheetQuery │  │  Cache  │  │ Google Planilha │  ││
│  │  └─────────┘  └────────────┘  └─────────┘  └─────────────────┘  ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

---

## Arquivo Principal: Code.gs

```javascript
/**
 * CrawDebate - Code.gs
 * Entry point principal do sistema
 * Expõe as funções doGet() e doPost() para a Web App
 */

/**
 * Handler GET - Renderiza a interface HTML
 * @param {Object} e - Parâmetros da requisição
 * @return {HtmlOutput} Página HTML renderizada
 */
function doGet(e) {
  var page = e.parameter.page || 'index';
  var template = HtmlService.createTemplateFromFile(page);
  
  // Passa dados iniciais para o template
  template.user = getCurrentUser();
  template.config = getPublicConfig();
  
  return template.evaluate()
    .setTitle('CrawDebate - Sistema de Debates')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

/**
 * Handler POST - Processa requisições da API REST
 * @param {Object} e - Dados da requisição POST
 * @return {TextOutput} Resposta JSON
 */
function doPost(e) {
  try {
    var data = JSON.parse(e.postData.contents);
    var action = data.action;
    var payload = data.payload;
    
    var result = APIRouter.route(action, payload);
    
    return ContentService
      .createTextOutput(JSON.stringify(result))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (error) {
    return ContentService
      .createTextOutput(JSON.stringify({
        success: false,
        error: error.message
      }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

/**
 * Inclui arquivos HTML parciais
 * @param {string} filename - Nome do arquivo
 * @return {string} Conteúdo HTML
 */
function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent();
}
```

---

## Funções do Backend Expostas ao Frontend

### DebateService.gs

```javascript
/**
 * CrawDebate - DebateService.gs
 * Serviços de Debate expostos ao frontend
 * Estas funções são chamadas via google.script.run
 */

/**
 * Lista todos os debates com paginação
 * @param {Object} options - Opções de listagem
 * @return {Object} Lista de debates e metadados
 */
function getDebates(options) {
  options = options || {};
  var page = options.page || 1;
  var limit = options.limit || 10;
  var categoryId = options.categoryId || null;
  var status = options.status || 'active';
  
  try {
    // Verifica cache primeiro
    var cacheKey = 'debates_' + page + '_' + limit + '_' + categoryId;
    var cached = Cache.get(cacheKey);
    if (cached) return cached;
    
    // Busca do banco de dados (Google Planilha)
    var debates = Sheets.query('Debates', {
      where: { status: status, categoryId: categoryId },
      orderBy: { createdAt: 'DESC' },
      limit: limit,
      offset: (page - 1) * limit
    });
    
    // Enriquece com dados do autor
    debates = debates.map(function(debate) {
      debate.author = Sheets.findById('Users', debate.authorId);
      debate.category = Sheets.findById('Categories', debate.categoryId);
      debate.commentCount = Sheets.count('Comments', { debateId: debate.id });
      return debate;
    });
    
    var total = Sheets.count('Debates', { status: status });
    
    var result = {
      success: true,
      data: debates,
      pagination: {
        page: page,
        limit: limit,
        total: total,
        pages: Math.ceil(total / limit)
      }
    };
    
    // Salva em cache por 5 minutos
    Cache.set(cacheKey, result, 300);
    
    return result;
  } catch (error) {
    ErrorHandler.log(error, 'getDebates');
    return { success: false, error: error.message };
  }
}

/**
 * Obtém um debate por ID
 * @param {string} debateId - ID do debate
 * @return {Object} Dados do debate
 */
function getDebateById(debateId) {
  try {
    var debate = Sheets.findById('Debates', debateId);
    if (!debate) {
      return { success: false, error: 'Debate não encontrado' };
    }
    
    // Incrementa visualizações
    Sheets.update('Debates', debateId, { 
      viewCount: debate.viewCount + 1 
    });
    
    // Enriquece com dados relacionados
    debate.author = Sheets.findById('Users', debate.authorId);
    debate.category = Sheets.findById('Categories', debate.categoryId);
    debate.tags = getDebateTags(debateId);
    debate.comments = getDebateComments(debateId);
    debate.votes = getDebateVotes(debateId);
    
    return { success: true, data: debate };
  } catch (error) {
    ErrorHandler.log(error, 'getDebateById');
    return { success: false, error: error.message };
  }
}

/**
 * Cria um novo debate
 * @param {Object} debateData - Dados do debate
 * @return {Object} Debate criado
 */
function createDebate(debateData) {
  try {
    // Verifica autenticação
    var user = getCurrentUser();
    if (!user) {
      return { success: false, error: 'Usuário não autenticado' };
    }
    
    // Valida dados
    var validation = DebateValidator.validate(debateData);
    if (!validation.valid) {
      return { success: false, error: validation.errors };
    }
    
    // Moderação com Gemini (opcional)
    var moderation = GeminiModeration.check(debateData.content);
    if (moderation.flagged) {
      return { success: false, error: 'Conteúdo não permitido', flags: moderation.flags };
    }
    
    // Cria o debate
    var debate = {
      id: Utilities.getUuid(),
      title: debateData.title,
      description: debateData.description,
      categoryId: debateData.categoryId,
      authorId: user.id,
      status: 'active',
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
      viewCount: 0,
      commentCount: 0,
      voteCount: 0
    };
    
    Sheets.insert('Debates', debate);
    
    // Notifica seguidores do autor
    NotificationService.notifyFollowers(user.id, 'new_debate', debate);
    
    // Invalida cache
    Cache.deletePattern('debates_*');
    
    return { success: true, data: debate };
  } catch (error) {
    ErrorHandler.log(error, 'createDebate');
    return { success: false, error: error.message };
  }
}

/**
 * Vota em um debate
 * @param {string} debateId - ID do debate
 * @param {string} voteType - Tipo de voto ('up' ou 'down')
 * @return {Object} Resultado da votação
 */
function voteDebate(debateId, voteType) {
  try {
    var user = getCurrentUser();
    if (!user) {
      return { success: false, error: 'Usuário não autenticado' };
    }
    
    // Verifica se já votou
    var existingVote = Sheets.findOne('Votes', {
      userId: user.id,
      targetType: 'debate',
      targetId: debateId
    });
    
    if (existingVote) {
      // Atualiza voto existente
      Sheets.update('Votes', existingVote.id, { voteType: voteType });
    } else {
      // Cria novo voto
      Sheets.insert('Votes', {
        id: Utilities.getUuid(),
        userId: user.id,
        targetType: 'debate',
        targetId: debateId,
        voteType: voteType,
        createdAt: new Date().toISOString()
      });
    }
    
    // Atualiza contador no debate
    var voteCount = Sheets.count('Votes', { targetType: 'debate', targetId: debateId });
    Sheets.update('Debates', debateId, { voteCount: voteCount });
    
    return { success: true, voteCount: voteCount };
  } catch (error) {
    ErrorHandler.log(error, 'voteDebate');
    return { success: false, error: error.message };
  }
}
```

---

## Frontend: Template HTML com Integração

### debate-app.html

```html
<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>CrawDebate</title>
  <?!= include('neon-styles'); ?>
</head>
<body>
  <!-- Header -->
  <header class="header">
    <button class="menu-toggle" onclick="UI.toggleMenu()">
      <span class="material-icons-outlined">menu</span>
    </button>
    <div class="logo font-orbitron text-neon-green">CRAWDEBATE</div>
    <div class="user-area" id="user-area"></div>
  </header>

  <!-- Loading Overlay -->
  <div class="gas-overlay" id="loading-overlay" style="display:none;">
    <div class="gas-loader">
      <div class="pulse-ring"></div>
      <span class="font-orbitron">CARREGANDO...</span>
    </div>
  </div>

  <!-- Main Content -->
  <main class="main" id="main-content">
    <!-- Lista de Debates -->
    <div id="debates-container">
      <div class="section-header">
        <h2 class="font-orbitron text-neon-purple">Debates Ativos</h2>
        <button class="btn btn-primary" onclick="DebateUI.showCreateModal()">
          <span class="material-icons-outlined">add</span> Novo Debate
        </button>
      </div>
      <div id="debates-list" class="debates-grid"></div>
      <div id="pagination"></div>
    </div>

    <!-- Detalhe do Debate (inicialmente oculto) -->
    <div id="debate-detail" style="display:none;"></div>
  </main>

  <!-- Modal: Criar Debate -->
  <div class="modal" id="create-debate-modal">
    <div class="modal-content card">
      <div class="modal-header">
        <h3 class="font-orbitron">Criar Novo Debate</h3>
        <button class="close-btn" onclick="UI.closeModal('create-debate-modal')">&times;</button>
      </div>
      <form id="create-debate-form" onsubmit="DebateUI.handleCreateSubmit(event)">
        <div class="form-group">
          <label>Título</label>
          <input type="text" name="title" class="input" required placeholder="Título do debate...">
        </div>
        <div class="form-group">
          <label>Categoria</label>
          <select name="categoryId" class="input" id="category-select"></select>
        </div>
        <div class="form-group">
          <label>Descrição</label>
          <textarea name="description" class="input" rows="5" required 
                    placeholder="Descreva o tema do debate..."></textarea>
        </div>
        <div class="form-actions">
          <button type="button" class="btn btn-secondary" 
                  onclick="UI.closeModal('create-debate-modal')">Cancelar</button>
          <button type="submit" class="btn btn-primary">
            <span class="material-icons-outlined">send</span> Publicar
          </button>
        </div>
      </form>
    </div>
  </div>

  <?!= include('neon-scripts'); ?>
  <?!= include('debate-scripts'); ?>
</body>
</html>
```

---

## Frontend: JavaScript de Integração

### debate-scripts.html

```html
<script>
/**
 * CrawDebate - Frontend JavaScript
 * Integração com Backend via google.script.run
 */

// ============================================
// Namespace de Serviços (Comunicação Backend)
// ============================================
var DebateService = {
  
  /**
   * Carrega lista de debates
   */
  loadDebates: function(options) {
    return new Promise(function(resolve, reject) {
      UI.showLoading();
      
      google.script.run
        .withSuccessHandler(function(result) {
          UI.hideLoading();
          if (result.success) {
            resolve(result);
          } else {
            reject(new Error(result.error));
          }
        })
        .withFailureHandler(function(error) {
          UI.hideLoading();
          reject(error);
        })
        .getDebates(options);
    });
  },
  
  /**
   * Carrega um debate específico
   */
  loadDebate: function(debateId) {
    return new Promise(function(resolve, reject) {
      UI.showLoading();
      
      google.script.run
        .withSuccessHandler(function(result) {
          UI.hideLoading();
          if (result.success) {
            resolve(result.data);
          } else {
            reject(new Error(result.error));
          }
        })
        .withFailureHandler(function(error) {
          UI.hideLoading();
          reject(error);
        })
        .getDebateById(debateId);
    });
  },
  
  /**
   * Cria novo debate
   */
  create: function(debateData) {
    return new Promise(function(resolve, reject) {
      UI.showLoading('Criando debate...');
      
      google.script.run
        .withSuccessHandler(function(result) {
          UI.hideLoading();
          if (result.success) {
            resolve(result.data);
          } else {
            reject(new Error(result.error));
          }
        })
        .withFailureHandler(function(error) {
          UI.hideLoading();
          reject(error);
        })
        .createDebate(debateData);
    });
  },
  
  /**
   * Vota em um debate
   */
  vote: function(debateId, voteType) {
    return new Promise(function(resolve, reject) {
      google.script.run
        .withSuccessHandler(function(result) {
          if (result.success) {
            resolve(result);
          } else {
            reject(new Error(result.error));
          }
        })
        .withFailureHandler(reject)
        .voteDebate(debateId, voteType);
    });
  }
};

// ============================================
// Namespace de UI de Debates
// ============================================
var DebateUI = {
  currentPage: 1,
  debates: [],
  
  /**
   * Inicializa a interface de debates
   */
  init: function() {
    this.loadCategories();
    this.loadDebates();
  },
  
  /**
   * Carrega categorias para o select
   */
  loadCategories: function() {
    google.script.run
      .withSuccessHandler(function(result) {
        if (result.success) {
          var select = document.getElementById('category-select');
          select.innerHTML = result.data.map(function(cat) {
            return '<option value="' + cat.id + '">' + cat.name + '</option>';
          }).join('');
        }
      })
      .getCategories();
  },
  
  /**
   * Carrega e renderiza lista de debates
   */
  loadDebates: function(page) {
    var self = this;
    page = page || 1;
    
    DebateService.loadDebates({ page: page, limit: 10 })
      .then(function(result) {
        self.debates = result.data;
        self.currentPage = page;
        self.renderDebatesList(result.data);
        self.renderPagination(result.pagination);
      })
      .catch(function(error) {
        UI.showAlert('Erro ao carregar debates: ' + error.message, 'error');
      });
  },
  
  /**
   * Renderiza a lista de debates
   */
  renderDebatesList: function(debates) {
    var container = document.getElementById('debates-list');
    
    if (debates.length === 0) {
      container.innerHTML = '<div class="empty-state">Nenhum debate encontrado</div>';
      return;
    }
    
    container.innerHTML = debates.map(function(debate) {
      return `
        <div class="card debate-card" onclick="DebateUI.showDebate('${debate.id}')">
          <div class="debate-category badge badge-primary">${debate.category.name}</div>
          <h3 class="debate-title">${debate.title}</h3>
          <p class="debate-excerpt">${debate.description.substring(0, 150)}...</p>
          <div class="debate-meta">
            <span class="author">
              <span class="material-icons-outlined">person</span>
              ${debate.author.username}
            </span>
            <span class="stats">
              <span class="material-icons-outlined">visibility</span> ${debate.viewCount}
              <span class="material-icons-outlined">comment</span> ${debate.commentCount}
              <span class="material-icons-outlined">thumb_up</span> ${debate.voteCount}
            </span>
          </div>
          <div class="debate-date">${UI.formatDate(debate.createdAt)}</div>
        </div>
      `;
    }).join('');
  },
  
  /**
   * Renderiza paginação
   */
  renderPagination: function(pagination) {
    var container = document.getElementById('pagination');
    var pages = [];
    
    for (var i = 1; i <= pagination.pages; i++) {
      var activeClass = i === pagination.page ? 'active' : '';
      pages.push('<button class="page-btn ' + activeClass + '" onclick="DebateUI.loadDebates(' + i + ')">' + i + '</button>');
    }
    
    container.innerHTML = '<div class="pagination">' + pages.join('') + '</div>';
  },
  
  /**
   * Exibe modal de criação
   */
  showCreateModal: function() {
    document.getElementById('create-debate-form').reset();
    UI.openModal('create-debate-modal');
  },
  
  /**
   * Processa submissão do formulário de criação
   */
  handleCreateSubmit: function(event) {
    event.preventDefault();
    var form = event.target;
    var formData = new FormData(form);
    
    var debateData = {
      title: formData.get('title'),
      categoryId: formData.get('categoryId'),
      description: formData.get('description')
    };
    
    DebateService.create(debateData)
      .then(function(debate) {
        UI.closeModal('create-debate-modal');
        UI.showAlert('Debate criado com sucesso!', 'success');
        DebateUI.loadDebates(1); // Recarrega lista
      })
      .catch(function(error) {
        UI.showAlert('Erro: ' + error.message, 'error');
      });
  },
  
  /**
   * Exibe detalhes de um debate
   */
  showDebate: function(debateId) {
    var self = this;
    
    DebateService.loadDebate(debateId)
      .then(function(debate) {
        self.renderDebateDetail(debate);
        document.getElementById('debates-container').style.display = 'none';
        document.getElementById('debate-detail').style.display = 'block';
      })
      .catch(function(error) {
        UI.showAlert('Erro ao carregar debate: ' + error.message, 'error');
      });
  },
  
  /**
   * Renderiza detalhes do debate
   */
  renderDebateDetail: function(debate) {
    var container = document.getElementById('debate-detail');
    
    container.innerHTML = `
      <button class="btn btn-secondary" onclick="DebateUI.backToList()">
        <span class="material-icons-outlined">arrow_back</span> Voltar
      </button>
      
      <article class="debate-full card">
        <header class="debate-header">
          <span class="badge badge-primary">${debate.category.name}</span>
          <h1 class="debate-title font-orbitron">${debate.title}</h1>
          <div class="debate-author">
            Por <strong>${debate.author.username}</strong> em ${UI.formatDate(debate.createdAt)}
          </div>
        </header>
        
        <div class="debate-content">
          ${debate.description}
        </div>
        
        <div class="debate-actions">
          <button class="btn vote-btn" onclick="DebateUI.vote('${debate.id}', 'up')">
            <span class="material-icons-outlined">thumb_up</span>
            <span id="vote-count">${debate.voteCount}</span>
          </button>
          <button class="btn vote-btn" onclick="DebateUI.vote('${debate.id}', 'down')">
            <span class="material-icons-outlined">thumb_down</span>
          </button>
          <button class="btn" onclick="DebateUI.toggleCommentForm()">
            <span class="material-icons-outlined">comment</span> Comentar
          </button>
        </div>
        
        <section class="comments-section">
          <h3>Comentários (${debate.comments.length})</h3>
          <div id="comments-list">
            ${this.renderComments(debate.comments)}
          </div>
        </section>
      </article>
    `;
  },
  
  /**
   * Renderiza lista de comentários
   */
  renderComments: function(comments) {
    if (!comments || comments.length === 0) {
      return '<p class="empty-comments">Nenhum comentário ainda. Seja o primeiro!</p>';
    }
    
    return comments.map(function(comment) {
      return `
        <div class="comment-card card">
          <div class="comment-header">
            <strong>${comment.author.username}</strong>
            <span class="comment-date">${UI.formatDate(comment.createdAt)}</span>
          </div>
          <div class="comment-content">${comment.content}</div>
          <div class="comment-actions">
            <button class="btn-icon" onclick="CommentUI.vote('${comment.id}', 'up')">
              <span class="material-icons-outlined">thumb_up</span> ${comment.voteCount}
            </button>
            <button class="btn-icon" onclick="CommentUI.reply('${comment.id}')">
              <span class="material-icons-outlined">reply</span> Responder
            </button>
          </div>
        </div>
      `;
    }).join('');
  },
  
  /**
   * Processa votação
   */
  vote: function(debateId, voteType) {
    DebateService.vote(debateId, voteType)
      .then(function(result) {
        document.getElementById('vote-count').textContent = result.voteCount;
        UI.showAlert('Voto registrado!', 'success');
      })
      .catch(function(error) {
        UI.showAlert('Erro ao votar: ' + error.message, 'error');
      });
  },
  
  /**
   * Volta para a lista de debates
   */
  backToList: function() {
    document.getElementById('debate-detail').style.display = 'none';
    document.getElementById('debates-container').style.display = 'block';
  }
};

// ============================================
// Namespace de UI Geral
// ============================================
var UI = {
  
  showLoading: function(message) {
    var overlay = document.getElementById('loading-overlay');
    if (message) {
      overlay.querySelector('.font-orbitron').textContent = message;
    }
    overlay.style.display = 'flex';
  },
  
  hideLoading: function() {
    document.getElementById('loading-overlay').style.display = 'none';
  },
  
  openModal: function(modalId) {
    document.getElementById(modalId).classList.add('active');
  },
  
  closeModal: function(modalId) {
    document.getElementById(modalId).classList.remove('active');
  },
  
  showAlert: function(message, type) {
    // Implementa toast/alert notification
    var alert = document.createElement('div');
    alert.className = 'alert alert-' + type;
    alert.innerHTML = message;
    document.body.appendChild(alert);
    
    setTimeout(function() {
      alert.classList.add('show');
    }, 100);
    
    setTimeout(function() {
      alert.classList.remove('show');
      setTimeout(function() {
        alert.remove();
      }, 300);
    }, 3000);
  },
  
  formatDate: function(dateStr) {
    var date = new Date(dateStr);
    return date.toLocaleDateString('pt-BR') + ' às ' + date.toLocaleTimeString('pt-BR', { hour: '2-digit', minute: '2-digit' });
  },
  
  toggleMenu: function() {
    document.getElementById('side-menu').classList.toggle('active');
    document.querySelector('.overlay').classList.toggle('active');
  }
};

// ============================================
// Inicialização
// ============================================
document.addEventListener('DOMContentLoaded', function() {
  DebateUI.init();
});
</script>
```

---

## Fluxo de Dados Completo

### 1. Listar Debates

```
Frontend                          Backend
   │                                 │
   │  1. DebateUI.loadDebates()      │
   ├──────────────────────────────►  │
   │     google.script.run           │
   │       .getDebates({page:1})     │
   │                                 │
   │                          ┌──────┴──────┐
   │                          │  getDebates │
   │                          │  - Cache    │
   │                          │  - Sheets   │
   │                          └──────┬──────┘
   │                                 │
   │  2. withSuccessHandler(result)  │
   │◄──────────────────────────────  │
   │                                 │
   │  3. renderDebatesList(data)     │
   │                                 │
```

### 2. Criar Debate

```
Frontend                          Backend
   │                                 │
   │  1. handleCreateSubmit()        │
   │     DebateService.create()      │
   ├──────────────────────────────►  │
   │     google.script.run           │
   │       .createDebate(data)       │
   │                                 │
   │                          ┌──────┴──────────────────┐
   │                          │  createDebate           │
   │                          │  - getCurrentUser()     │
   │                          │  - DebateValidator      │
   │                          │  - GeminiModeration     │
   │                          │  - Sheets.insert()      │
   │                          │  - NotificationService  │
   │                          │  - Cache.delete()       │
   │                          └──────┬──────────────────┘
   │                                 │
   │  2. withSuccessHandler(result)  │
   │◄──────────────────────────────  │
   │                                 │
   │  3. showAlert('Sucesso!')       │
   │     loadDebates() // refresh    │
   │                                 │
```

---

## Resumo das Tecnologias

| Camada | Tecnologia | Responsabilidade |
|--------|------------|------------------|
| **Frontend** | HTML5, CSS3 (Neon), JS | Interface do usuário |
| **Bridge** | `google.script.run` | Comunicação assíncrona |
| **Entry Points** | `doGet()`, `doPost()` | Recebe requisições web |
| **API** | APIRouter, APIMiddleware | Roteamento e validação |
| **Business** | Debate, Comment, Vote, User | Lógica de negócio |
| **Data** | Sheets, Cache | Persistência em Google Planilhas |
| **AI** | Gemini API | Moderação e análise |

---

## Boas Práticas Implementadas

1. **Promises no Frontend** - Encapsula `google.script.run` em Promises para código mais limpo
2. **Namespaces** - Organização de código em módulos (`DebateService`, `DebateUI`, `UI`)
3. **Cache** - Reduz chamadas ao banco de dados
4. **Validação Dupla** - Frontend (UX) e Backend (segurança)
5. **Error Handling** - Tratamento consistente de erros
6. **Loading States** - Feedback visual durante operações
7. **Modularização** - Separação de concerns entre camadas
