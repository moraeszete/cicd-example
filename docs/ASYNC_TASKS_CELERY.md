# 🚀 ASYNC TASKS COM CELERY - GUIA COMPLETO

**Data:** 29 de Janeiro de 2026  
**Versão:** 1.0  
**Status:** Implementação em Progresso

---

## 📋 ÍNDICE

1. [O que é Celery?](#o-que-é-celery)
2. [Arquitetura Async em Python](#arquitetura-async-em-python)
3. [Setup Celery no BuildX](#setup-celery-no-buildx)
4. [Exemplos Práticos](#exemplos-práticos)
5. [Implementação em Operações Pesadas](#implementação-em-operações-pesadas)
6. [Monitoramento](#monitoramento)

---

## O que é Celery?

### 📌 Conceito

**Celery** é uma biblioteca Python que permite executar tarefas **assincronamente** usando filas de mensagens.

```
┌─────────────────────────────────────────────────────────────┐
│                    SYNC vs ASYNC                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ SÍNCRONO (Bloqueante):                                       │
│ Usuario envía request → Django processa → Usuario espera     │
│ ❌ Timeout se demorar > 30s                                  │
│                                                              │
│ ASSÍNCRONO (Não-bloqueante):                                 │
│ Usuario envía request → Celery recebe → Retorna imediato     │
│ Celery processa em background (minutos/horas)               │
│ ✅ Usuario continua usando app                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 🎯 Quando usar Celery

✅ **USE CELERY:**
- Importação de dados (> 1 segundo)
- Integração com sistemas externos (TOTVS, Steel)
- Processamento de PDF/Excel
- Envio de emails em massa
- Relatórios pesados
- Sincronização de banco dados

❌ **NÃO USE CELERY:**
- Operações < 100ms
- Queries simples de BD
- Cálculos triviais

---

## Arquitetura Async em Python

### 📊 Componentes

```
┌────────────────────────────────────────────────────────────────┐
│                      DJANGO (REST API)                         │
│                                                                │
│  GET /api/bx/st-pecas/importa/                                │
│    └─ task.delay()  ─────────────────────────┐               │
└────────────────────────────────────────────────────────────────┘
                                                │
                                                ▼
┌────────────────────────────────────────────────────────────────┐
│                    MESSAGE BROKER (REDIS)                      │
│                                                                │
│  Queue: [task_id_1, task_id_2, task_id_3, ...]                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                                                │
                                                ▼
┌────────────────────────────────────────────────────────────────┐
│                  CELERY WORKERS (Background)                   │
│                                                                │
│  Worker 1: Processando task_id_1 (importaSteel)               │
│  Worker 2: Processando task_id_2 (geraExcel)                  │
│  Worker 3: Processando task_id_3 (enviaPDF)                   │
│                                                                │
│  [Rodando 24/7 em processo separado]                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                                                │
                                                ▼
┌────────────────────────────────────────────────────────────────┐
│                  RESULT BACKEND (REDIS)                        │
│                                                                │
│  task_id_1: {"status": "SUCCESS", "result": {...}}            │
│  task_id_2: {"status": "PENDING", "progress": 45%}            │
│  task_id_3: {"status": "FAILURE", "error": "..."}             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 🔄 Fluxo de Execução

```python
# Fluxo SÍNCRONO (Bloqueante)
@view
def importa_sync(request):
    # Cliente espera enquanto rodando
    resultado = importa_steel()  # 10 minutos!!!
    return resultado

# Fluxo ASSÍNCRONO (Não-bloqueante)
@view
def importa_async(request):
    # Retorna imediato
    task = importa_steel.delay()  # Enfileira
    return {"task_id": task.id}   # Cliente continua

# Enquanto isso, Celery processa em background...
```

---

## Setup Celery no BuildX

### 🔧 Status Atual

O projeto **JÁ TEM CELERY CONFIGURADO:**

**Arquivo:** `core/project/settings/base.py` (Linhas 127-133)

```python
# Celery Configuration
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'America/Sao_Paulo'
```

✅ **Já tem Redis!**  
✅ **Já tem Celery!**  
❌ **Mas NÃO está sendo USADO!**

### ✅ Iniciar Celery Worker

```bash
# Em um terminal SEPARADO (não no mesmo do Django)
cd c:/Projetos/Metasa/buildx-back-py
poetry run celery -A core.project worker --loglevel=info

# Esperado:
# -------------- celery@LAPTOP v5.5.1
#  --- ***** -----
# -- ******* ----
# - *** --- * ---
# - ** ---------- [config]
# - ** ---------- .
# - ** ---------- [queues]
# - ** ---------- .celery.pidbox
# - *** --- * --- [tasks]
```

---

## Exemplos Práticos

### Exemplo 1: Task Simples (Hello World)

**Arquivo:** `core/general/tasks.py` (Criar novo)

```python
from celery import shared_task
import logging

logger = logging.getLogger(__name__)

@shared_task(bind=True)
def hello_world(self):
    """
    Task mais simples possível
    
    Uso:
        hello_world.delay()
    """
    logger.info(f"Task ID: {self.request.id}")
    return "Hello, Async World!"
```

**Uso em View:**

```python
from core.general.tasks import hello_world

def minha_view(request):
    # Enfileira task e retorna imediato
    task = hello_world.delay()
    return {"task_id": task.id, "status": "queued"}
```

**Resultado:**

```bash
# No console Celery:
# [2026-01-29 10:15:30,123] Task ID: abc123xyz
# [2026-01-29 10:15:30,125] Result: Hello, Async World!
```

---

### Exemplo 2: Task com Retry (Robustez)

**Arquivo:** `core/general/tasks.py`

```python
from celery import shared_task
from celery.exceptions import MaxRetriesExceededError
import requests
import logging

logger = logging.getLogger(__name__)

@shared_task(
    bind=True,
    max_retries=3,  # Máximo de tentativas
    default_retry_delay=60  # Aguarda 60s antes de retry
)
def sync_with_external_api(self, item_id):
    """
    Task que tenta 3 vezes se falhar
    
    Uso:
        sync_with_external_api.delay(123)
    """
    try:
        logger.info(f"Sincronizando item {item_id} (tentativa {self.request.retries + 1})")
        
        # Simula chamada a API externa
        response = requests.get(f"https://api.totvs.com/items/{item_id}", timeout=5)
        response.raise_for_status()
        
        return {"status": "success", "data": response.json()}
        
    except requests.exceptions.Timeout:
        logger.warning(f"Timeout ao sincronizar item {item_id}")
        # Retry com backoff exponencial
        raise self.retry(exc=Timeout, countdown=60 * (2 ** self.request.retries))
        
    except requests.exceptions.ConnectionError as exc:
        logger.error(f"Erro de conexão: {exc}")
        try:
            # Retry até 3 vezes
            raise self.retry(exc=exc, countdown=120)
        except MaxRetriesExceededError:
            logger.critical(f"Falha permanente ao sincronizar {item_id}")
            return {"status": "failed", "error": str(exc)}
```

---

### Exemplo 3: Task com Progresso (Monitor)

**Arquivo:** `core/general/tasks.py`

```python
from celery import shared_task

@shared_task(bind=True)
def processar_lista_grande(self, ids_list):
    """
    Task que reporta progresso para o front-end
    
    Uso:
        task = processar_lista_grande.delay([1,2,3,4,5])
        # Após 10s: task.state == "PROGRESS", task.info == {"current": 3, "total": 5}
        # Após 20s: task.state == "SUCCESS", task.result == {...}
    """
    total_items = len(ids_list)
    
    resultados = []
    
    for i, item_id in enumerate(ids_list, 1):
        # Processa item
        resultado = processar_item(item_id)
        resultados.append(resultado)
        
        # 🔴 REPORTA PROGRESSO (front-end fica informado)
        self.update_state(
            state='PROGRESS',
            meta={
                'current': i,
                'total': total_items,
                'percent': (i / total_items) * 100
            }
        )
    
    return {
        "status": "completed",
        "total_processados": total_items,
        "resultados": resultados
    }

def processar_item(item_id):
    """Simula processamento de 1 item"""
    import time
    time.sleep(2)  # Simula trabalho
    return {"item_id": item_id, "status": "ok"}
```

**Como monitorar no Front-end (JavaScript):**

```javascript
// Após chamar /api/tasks/processar/?ids=1,2,3,4,5 e receber task_id
function monitorarProgresso(taskId) {
    const checkProgress = setInterval(() => {
        fetch(`/api/tasks/${taskId}/status/`)
            .then(r => r.json())
            .then(data => {
                if (data.state === 'PROGRESS') {
                    // 45% pronto
                    document.getElementById('progress').value = data.meta.percent;
                } else if (data.state === 'SUCCESS') {
                    // Pronto!
                    console.log("Resultado:", data.result);
                    clearInterval(checkProgress);
                } else if (data.state === 'FAILURE') {
                    // Erro!
                    console.error("Erro:", data.error);
                    clearInterval(checkProgress);
                }
            });
    }, 1000);  // Check a cada 1 segundo
}
```

---

### Exemplo 4: Scheduled Tasks (CRON)

**Arquivo:** `core/general/tasks.py`

```python
from celery import shared_task
from celery.schedules import crontab
from django.conf import settings

@shared_task
def limpar_logs_antigos():
    """
    Executa AUTOMATICAMENTE todo dia às 2:00 AM
    """
    import os
    from pathlib import Path
    
    logs_dir = Path(settings.BASE_DIR) / "logs"
    
    # Remove logs com > 30 dias
    for arquivo in logs_dir.glob("*.log"):
        if arquivo.stat().st_mtime < (time.time() - 30 * 86400):
            arquivo.unlink()
    
    return "Logs antigos removidos"

# Configurar schedule em settings/base.py
# CELERY_BEAT_SCHEDULE = {
#     'limpar-logs': {
#         'task': 'core.general.tasks.limpar_logs_antigos',
#         'schedule': crontab(hour=2, minute=0),  # 2:00 AM
#     },
# }
```

---

## Implementação em Operações Pesadas

### 🎯 Caso 1: Importação de Steel

**Arquivo:** `core/general/tasks.py` (Adicionar)

```python
from celery import shared_task
from django.conf import settings
import logging

logger = logging.getLogger(__name__)

@shared_task(bind=True, max_retries=3)
def importa_steel_async(self, projeto, fase, etapa):
    """
    Task para importar peças do Steel SEM bloquear a API
    
    Uso em view:
        from core.general.tasks import importa_steel_async
        
        task = importa_steel_async.delay(projeto, fase, etapa)
        return {"task_id": task.id, "status": "processing"}
    
    Monitorar progresso:
        GET /api/tasks/{task_id}/status/
    """
    from core.general.services.steel_pecas_functions import importa_steel
    from core.general.utils.monitor.helper import MonitorHelperClass
    from core.general.exceptions import SteelEmptyReturn
    
    try:
        logger.info(f"[ASYNC] Iniciando importação: {projeto}-{fase}-{etapa}")
        
        # Reporta que iniciou
        self.update_state(
            state='PROGRESS',
            meta={
                'status': 'iniciando',
                'projeto': projeto,
                'fase': fase,
                'etapa': etapa
            }
        )
        
        # EXECUTA A IMPORTAÇÃO (pode levar minutos!)
        resultado = importa_steel(
            filter_query="",
            bx_projeto=projeto,
            bx_fase=fase,
            bx_etapa=etapa
        )
        
        logger.info(f"[ASYNC] Importação concluída: {projeto}-{fase}-{etapa}")
        
        return {
            "status": "success",
            "projeto": projeto,
            "fase": fase,
            "etapa": etapa,
            "resultado": resultado
        }
        
    except SteelEmptyReturn as exc:
        logger.warning(f"[ASYNC] Steel sem dados: {exc}")
        return {"status": "warning", "erro": str(exc)}
        
    except Exception as exc:
        logger.error(f"[ASYNC] Erro na importação: {exc}")
        # Retry com backoff
        try:
            raise self.retry(exc=exc, countdown=300)  # Tenta novamente em 5min
        except Exception:
            return {"status": "failed", "erro": str(exc)}
```

**Usar em View:**

```python
# File: core/apps/st_pecas/views.py

from core.general.tasks import importa_steel_async

class Bx_st_pecas_ViewSet(viewsets.ModelViewSet):
    
    def create(self, request):
        """Endpoint: POST /api/bx/st-pecas/"""
        
        projeto = request.data.get("projeto")
        fase = request.data.get("fase")
        etapa = request.data.get("etapa")
        
        # ✅ ASSÍNCRONO: Retorna imediato
        task = importa_steel_async.delay(projeto, fase, etapa)
        
        return success_response(
            key="import_task",
            data={
                "task_id": str(task.id),
                "status": "queued",
                "message": "Importação iniciada! Você pode acompanhar o progresso."
            },
            status=status.HTTP_202_ACCEPTED
        )
```

---

### 🎯 Caso 2: Exportação de Excel Pesada

**Arquivo:** `core/general/tasks.py` (Adicionar)

```python
from celery import shared_task
from django.core.files.storage import default_storage
from django.conf import settings
import logging
import os

logger = logging.getLogger(__name__)

@shared_task(bind=True)
def exporta_excel_async(self, mef_nam):
    """
    Task para exportar dados em Excel SEM bloquear API
    
    Uso:
        task = exporta_excel_async.delay("MEF-001")
        return {"task_id": task.id}
    """
    from core.general.services.excel.exportaExcel import exporta_corridas
    import tempfile
    
    try:
        logger.info(f"[ASYNC] Iniciando export Excel: {mef_nam}")
        
        self.update_state(
            state='PROGRESS',
            meta={'status': 'gerando_excel', 'mef': mef_nam}
        )
        
        # Cria arquivo temporário
        with tempfile.NamedTemporaryFile(delete=False, suffix='.xlsx') as tmp:
            # Exporta dados
            exporta_corridas(mef_nam, tmp.name)
            
            # Salva em storage (S3, local, etc)
            excel_path = f"exports/{mef_nam}_{self.request.id}.xlsx"
            with open(tmp.name, 'rb') as f:
                default_storage.save(excel_path, f)
            
            # Remove arquivo temporário
            os.unlink(tmp.name)
        
        logger.info(f"[ASYNC] Excel gerado: {excel_path}")
        
        return {
            "status": "success",
            "file_path": excel_path,
            "download_url": f"/media/{excel_path}"
        }
        
    except Exception as exc:
        logger.error(f"[ASYNC] Erro ao gerar Excel: {exc}")
        return {"status": "failed", "erro": str(exc)}
```

---

### 🎯 Caso 3: Sincronização com TOTVS

**Arquivo:** `core/general/tasks.py` (Adicionar)

```python
from celery import shared_task
import logging

logger = logging.getLogger(__name__)

@shared_task(bind=True, max_retries=5)
def sincroniza_totvs_async(self, ordem_id):
    """
    Task para sincronizar ordem com TOTVS
    Com retry automático se falhar
    
    Uso:
        task = sincroniza_totvs_async.delay(ordem_id=123)
    """
    from core.general.services.erp.totvs.etapa.requests import AprovaEtapa
    from core.apps.st_ord_prod.models import Bx_st_ord_prod
    
    try:
        logger.info(f"[ASYNC] Sincronizando ordem {ordem_id} com TOTVS")
        
        # Busca ordem
        ordem = Bx_st_ord_prod.objects.get(id=ordem_id)
        
        # Aprova em TOTVS
        resultado = AprovaEtapa(ordem.mef_nam).fazer_request()
        
        # Atualiza BD local
        ordem.status_erp = 3  # Sucesso
        ordem.save()
        
        logger.info(f"[ASYNC] Ordem {ordem_id} sincronizada com sucesso")
        
        return {"status": "success", "ordem_id": ordem_id}
        
    except Exception as exc:
        logger.warning(f"[ASYNC] Erro ao sincronizar {ordem_id}: {exc}")
        # Retry com backoff exponencial: 60s, 120s, 240s, 480s, 960s
        countdown = 60 * (2 ** self.request.retries)
        raise self.retry(exc=exc, countdown=countdown)
```

---

## Monitoramento

### 📊 Visualizar Tasks em Execução

```bash
# Terminal 1: Django
poetry run python -m core.manage runserver

# Terminal 2: Celery Worker (verificar tasks)
poetry run celery -A core.project worker --loglevel=info

# Terminal 3: Monitor (Flower - interface web)
poetry run celery -A core.project flower

# Abrir navegador:
# http://localhost:5555
```

### 🔍 Flower Interface

**URL:** `http://localhost:5555`

```
Dashboard:
├── Tasks (ao vivo)
│   ├── importa_steel_async
│   │   ├── Status: PROGRESS (45%)
│   │   ├── Tempo decorrido: 2min 30s
│   │   ├── Worker: celery@laptop
│   │   └── Args: ("PRJ001", "A", "001")
│   │
│   ├── exporta_excel_async
│   │   ├── Status: SUCCESS (100%)
│   │   ├── Tempo total: 15s
│   │   └── Result: file_path: /media/exports/...
│   │
│   └── sync_com_api
│       ├── Status: FAILURE
│       ├── Error: ConnectionError
│       ├── Retries: 2/5
│       └── Próxima tentativa: em 4min

├── Workers (celery@laptop)
│   ├── CPU: 45%
│   ├── Memória: 120MB
│   ├── Tasks ativas: 3
│   └── Uptime: 12h 45m

└── Queues
    ├── default: 5 tasks pendentes
    ├── high_priority: 0 tasks
    └── low_priority: 12 tasks
```

---

### 📡 API para Monitorar Task Status

**Criar novo endpoint:**

```python
# File: core/general/views.py (Novo)

from rest_framework import viewsets
from rest_framework.decorators import action
from celery.result import AsyncResult
from core.general.utils.responses import success_response, error_response

class TaskStatusViewSet(viewsets.ViewSet):
    """
    Endpoint para monitorar status de tasks
    
    GET /api/tasks/{task_id}/status/
    Response:
    {
        "task_id": "abc123xyz",
        "state": "PROGRESS",
        "progress": 45,
        "result": {...}
    }
    """
    
    @action(detail=True, methods=['get'])
    def status(self, request, pk=None):
        """Retorna status de uma task"""
        task_id = pk
        task_result = AsyncResult(task_id)
        
        response_data = {
            "task_id": task_id,
            "state": task_result.state,
            "result": task_result.result,
        }
        
        if task_result.state == 'PROGRESS':
            response_data['progress'] = task_result.info.get('percent', 0)
            response_data['current'] = task_result.info.get('current')
            response_data['total'] = task_result.info.get('total')
        
        return success_response(key="task_status", data=response_data)

# Registrar em urls.py
router.register("tasks", TaskStatusViewSet, basename="task-status")
```

---

## Checklist de Implementação

### ✅ Setup Inicial

- [ ] Redis já está instalado? `redis-server --version`
- [ ] Celery já está em pyproject.toml? ✅ (Sim)
- [ ] Criar arquivo `core/general/tasks.py`
- [ ] Testar Celery worker:
  ```bash
  poetry run celery -A core.project worker --loglevel=info
  ```

### ✅ Implementar Tasks

- [ ] Importação de Steel (`importa_steel_async`)
- [ ] Exportação de Excel (`exporta_excel_async`)
- [ ] Sincronização TOTVS (`sincroniza_totvs_async`)
- [ ] Limpeza de logs (`limpar_logs_antigos`)

### ✅ Integrar com Views

- [ ] `st_pecas/views.py` - usar `importa_steel_async.delay()`
- [ ] `st_fab_job_marca/views.py` - usar exportação async
- [ ] `st_ord_prod/views.py` - usar sincronização async

### ✅ Monitorar

- [ ] Instalar Flower:
  ```bash
  poetry add flower
  ```
- [ ] Criar TaskStatusViewSet em views
- [ ] Testar endpoint `/api/tasks/{id}/status/`

---

## Comandos Úteis

```bash
# Iniciar worker
poetry run celery -A core.project worker --loglevel=info

# Monitorar (Flower)
poetry run celery -A core.project flower

# Ver fila pendente
poetry run celery -A core.project inspect active

# Limpar fila
poetry run celery -A core.project purge

# Worker com múltiplos processes
poetry run celery -A core.project worker --concurrency=4 --loglevel=info

# Testar task (Django shell)
poetry run python -m core.manage shell
>>> from core.general.tasks import hello_world
>>> task = hello_world.delay()
>>> task.id
'abc123xyz'
>>> task.status
'SUCCESS'
>>> task.result
'Hello, Async World!'
```

---

## 🎯 Ganhos Esperados

| Antes | Depois |
|-------|--------|
| Importar 1000 peças: **15 min** (timeout!) | **2 min** (async) |
| Gerar Excel: **30s** (usuário espera) | **5s** (retorna imediato) |
| Sincronizar TOTVS: **1 fail** | **5 retries automático** |
| CPU máxima: **95%** | **30-40%** (distribuído) |

---

**Gerado em:** 29/01/2026
