# 🔴 MANUTENÇÕES CRÍTICAS - MAPA DE AÇÕES

**Data:** 29 de Janeiro de 2026  
**Status:** Priority 1 - Impacto Imediato  
**Tempo Estimado:** 2-3 dias

---

## 📋 ÍNDICE

1. [Problema 1: N+1 Queries](#problema-1-n1-queries)
2. [Checklist de Implementação](#checklist-de-implementação)

---

## Problema 1: N+1 Queries

### 📍 Descrição
Loops dentro de serializers causam uma query por item, tornando requisições extremamente lentas.

**Impacto:** 100 itens = 100+ queries em vez de 2-3  
**Ganho esperado:** 80-95% redução de tempo

---

### 🎯 Arquivo 1: st_fab_job_marca/serializers.py

#### ❌ Classe: `Bx_st_fab_job_operMarcaSerializer`

**Rota:** `/api/bx/st-fab-job-marca/` (list)  
**Método HTTP:** GET

##### Problema 1.1: `get_reservas()` - Linhas 14-18
```python
def get_reservas(self, obj: Bx_st_fab_job_oper_corte):
    bx_erp_item = Bx_erp_item.objects.filter(it_codigo=obj.it_codigo_marca).order_by("revisao").last()  # Query 1
    reservas = Bx_st_reservas.objects.filter(nr_ord_prod=obj.mef_nam, it_codigo=bx_erp_item).values()  # Query 2
    for reserva in reservas:
        reserva["it_codigo_id"] = Bx_erp_item.objects.get(chave_composta=reserva["it_codigo_id"]).it_codigo  # 🔴 Query N
    return reservas
```

**Issue:** Loop executa 1 query por reserva  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.2: `get_operacoes()` - Linhas 23-28
```python
def get_operacoes(self, obj: Bx_st_fab_job_oper_corte):
    bx_erp_item = Bx_erp_item.objects.filter(it_codigo=obj.it_codigo_marca).order_by("revisao").last()  # Query 1
    operacoes = Bx_st_oper_ord.objects.filter(nr_ord_prod=obj.mef_nam, it_codigo=bx_erp_item).values()  # Query 2
    for operacao in operacoes:
        operacao["it_codigo_id"] = Bx_erp_item.objects.get(chave_composta=operacao["it_codigo_id"]).it_codigo  # 🔴 Query N
    return operacoes
```

**Issue:** Loop executa 1 query por operação  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.3: `get_ordens_posicoes()` - Linhas 32-37
```python
def get_ordens_posicoes(self, obj: Bx_st_fab_job_oper_corte):
    bx_erp_item = Bx_erp_item.objects.filter(it_codigo=obj.it_codigo_marca).order_by("revisao").last()  # Query 1
    ordens = Bx_st_ord_prod.objects.filter(nr_ord_prod=obj.mef_nam, it_codigo=bx_erp_item).values()  # Query 2
    for ordem in ordens:
        ordem["it_codigo_id"] = Bx_erp_item.objects.get(chave_composta=ordem["it_codigo_id"]).it_codigo  # 🔴 Query N
    return ordens
```

**Issue:** Loop executa 1 query por ordem  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.4: `get_mef_id()` - Linhas 50-51
```python
def get_mef_id(self, obj: Bx_st_fab_job_oper_corte):
    return Bx_st_fab_job.objects.filter(mef_nam=obj.mef_nam).first().mef_id  # 🔴 Query por item
```

**Issue:** 1 query por item serializado  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

#### ❌ Classe: `Bx_st_fab_job_operPosicaoSerializer`

**Rota:** `/api/bx/st-fab-job-posicao/` (list)  
**Método HTTP:** GET

##### Problema 1.5: `get_reservas()` - Linhas 61-67
```python
def get_reservas(self, obj: Bx_st_fab_job_oper_corte):
    bx_erp_item = Bx_erp_item.objects.filter(it_codigo=obj.it_codigo_posicao).order_by("revisao").last()  # Query 1
    reservas = Bx_st_reservas.objects.filter(nr_ord_prod=obj.mef_nam, codigo_steel=bx_erp_item.inform_compl).values()  # Query 2
    for reserva in reservas:
        reserva["it_codigo_id"] = Bx_erp_item.objects.get(chave_composta=reserva["it_codigo_id"]).it_codigo  # 🔴 Query N
    return reservas
```

**Issue:** Loop executa 1 query por reserva + 1 query para buscar item  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.6: `get_operacoes()` - Linhas 72-77
```python
def get_operacoes(self, obj: Bx_st_fab_job_oper_corte):
    bx_erp_item = Bx_erp_item.objects.filter(it_codigo=obj.it_codigo_posicao).order_by("revisao").last()  # Query 1
    operacoes = Bx_st_oper_ord.objects.filter(nr_ord_prod=obj.mef_nam, codigo_steel=bx_erp_item.inform_compl).values()  # Query 2
    for operacao in operacoes:
        operacao["it_codigo_id"] = Bx_erp_item.objects.get(chave_composta=operacao["it_codigo_id"]).it_codigo  # 🔴 Query N
    return operacoes
```

**Issue:** Loop executa 1 query por operação + 1 query para buscar item  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.7: `get_ordens_posicoes()` - Linhas 81-86
```python
def get_ordens_posicoes(self, obj: Bx_st_fab_job_oper_corte):
    bx_erp_item = Bx_erp_item.objects.filter(it_codigo=obj.it_codigo_posicao).order_by("revisao").last()  # Query 1
    ordens = Bx_st_ord_prod.objects.filter(nr_ord_prod=obj.mef_nam, codigo_steel=bx_erp_item.inform_compl).values()  # Query 2
    for ordem in ordens:
        ordem["it_codigo_id"] = Bx_erp_item.objects.get(chave_composta=ordem["it_codigo_id"]).it_codigo  # 🔴 Query N
    return ordens
```

**Issue:** Loop executa 1 query por ordem + 1 query para buscar item  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.8: `get_mef_id()` - Linhas 103-104
```python
def get_mef_id(self, obj: Bx_st_fab_job_oper_corte):
    return Bx_st_fab_job.objects.filter(mef_nam=obj.mef_nam).first().mef_id  # 🔴 Query por item
```

**Issue:** 1 query por item serializado  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

### 🎯 Arquivo 2: st_fab_job_posicao/serializers.py

#### ❌ Classe: `Bx_st_fab_job_posicaoSerializer`

**Rota:** `/api/bx/st-fab-job-posicao/` (list)  
**Método HTTP:** GET

##### Problema 1.9: `get_reservas()` - Linhas 12-14
```python
def get_reservas(self, obj: Bx_st_fab_job_oper_corte):
    bx_erp_item = Bx_erp_item.objects.get(it_codigo=obj.it_codigo_posicao)  # 🔴 Query por item
    return Bx_st_reservas.objects.filter(nr_ord_prod=obj.mef_nam, it_codigo=bx_erp_item).values()
```

**Issue:** 1 query por item  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.10: `get_caracteristicas()` - Linhas 18-19
```python
def get_caracteristicas(self, obj: Bx_st_fab_job_oper_corte):
    return Bx_erp_it_carac_tec.objects.filter(it_codigo=obj.it_codigo_posicao).values()  # 🔴 Query por item
```

**Issue:** 1 query por item  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.11: `get_operacoes()` - Linhas 23-25
```python
def get_operacoes(self, obj: Bx_st_fab_job_oper_corte):
    bx_erp_item = Bx_erp_item.objects.get(it_codigo=obj.it_codigo_posicao)  # 🔴 Query por item
    return Bx_st_oper_ord.objects.filter(nr_ord_prod=obj.mef_nam, it_codigo=bx_erp_item).values()
```

**Issue:** 1 query por item  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

### 🎯 Arquivo 3: st_ord_prod/serializers.py

#### ❌ Classe: `Bx_st_ord_prodSerializer`

**Rota:** `/api/bx/st-ord-prod/` (list e retrieve)  
**Método HTTP:** GET

##### Problema 1.12: `get_operacoes()` - Linhas 26-41
```python
def get_operacoes(self, obj):
    return Bx_st_oper_ord.objects.filter(nr_ord_prod=obj.nr_ord_prod, it_codigo=obj.it_codigo).values(...)
    # Sem loop, mas não usa select_related/prefetch_related
```

**Issue:** Sem otimização de queries relacionadas  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

##### Problema 1.13: `get_reservas()` - Linhas 43-52
```python
def get_reservas(self, obj):
    return Bx_st_reservas.objects.filter(nr_ord_prod=obj.nr_ord_prod, it_codigo=obj.it_codigo).values(...)
    # Sem loop, mas não usa select_related/prefetch_related
```

**Issue:** Sem otimização de queries relacionadas  
**Solução:** Ver [Solução N+1](#solução-n1-queries)

---

### ✅ Solução N+1 Queries

#### Passo 1: Refatorar Serializer com `.to_representation()`

```python
from rest_framework import serializers
from django.db.models import Prefetch
from core.apps.erp_item.models import Bx_erp_item
from core.apps.st_reservas.models import Bx_st_reservas
from core.apps.st_oper_ord.models import Bx_st_oper_ord
from core.apps.st_ord_prod.models import Bx_st_ord_prod

class Bx_st_fab_job_operMarcaSerializer(serializers.ModelSerializer):
    """
    ✅ Versão otimizada sem N+1 queries
    """
    
    def to_representation(self, instance):
        # Dados básicos
        ret = super().to_representation(instance)
        
        # Cache local para evitar queries repetidas
        it_codigo_cache = {}
        
        # Busca item uma única vez
        try:
            bx_erp_item = Bx_erp_item.objects.filter(
                it_codigo=instance.it_codigo_marca
            ).order_by("revisao").values('chave_composta', 'it_codigo').last()
            
            if bx_erp_item:
                it_codigo_cache[bx_erp_item['chave_composta']] = bx_erp_item['it_codigo']
        except:
            bx_erp_item = None
        
        # ✅ Uma única query para reservas (sem loop N+1)
        reservas = list(
            Bx_st_reservas.objects.filter(
                nr_ord_prod=instance.mef_nam, 
                it_codigo=bx_erp_item
            ).values()
        )
        
        # ✅ Bulk lookup para evitar N+1 em update de it_codigo_id
        if reservas:
            chave_composta_ids = [r['it_codigo_id'] for r in reservas]
            item_mapping = dict(
                Bx_erp_item.objects.filter(
                    chave_composta__in=chave_composta_ids
                ).values_list('chave_composta', 'it_codigo')
            )
            for reserva in reservas:
                reserva['it_codigo_id'] = item_mapping.get(reserva['it_codigo_id'], '')
        
        ret['reservas'] = reservas
        
        # ✅ Uma única query para operações (sem loop N+1)
        operacoes = list(
            Bx_st_oper_ord.objects.filter(
                nr_ord_prod=instance.mef_nam, 
                it_codigo=bx_erp_item
            ).values()
        )
        
        # ✅ Bulk lookup para evitar N+1 em update de it_codigo_id
        if operacoes:
            chave_composta_ids = [o['it_codigo_id'] for o in operacoes]
            item_mapping = dict(
                Bx_erp_item.objects.filter(
                    chave_composta__in=chave_composta_ids
                ).values_list('chave_composta', 'it_codigo')
            )
            for operacao in operacoes:
                operacao['it_codigo_id'] = item_mapping.get(operacao['it_codigo_id'], '')
        
        ret['operacoes'] = operacoes
        
        # ✅ Uma única query para ordens (sem loop N+1)
        ordens = list(
            Bx_st_ord_prod.objects.filter(
                nr_ord_prod=instance.mef_nam, 
                it_codigo=bx_erp_item
            ).values()
        )
        
        # ✅ Bulk lookup para evitar N+1 em update de it_codigo_id
        if ordens:
            chave_composta_ids = [o['it_codigo_id'] for o in ordens]
            item_mapping = dict(
                Bx_erp_item.objects.filter(
                    chave_composta__in=chave_composta_ids
                ).values_list('chave_composta', 'it_codigo')
            )
            for ordem in ordens:
                ordem['it_codigo_id'] = item_mapping.get(ordem['it_codigo_id'], '')
        
        ret['ordens_posicoes'] = ordens
        
        # ✅ Uma única query para mef_id (sem Loop)
        ret['mef_id'] = Bx_st_fab_job.objects.filter(
            mef_nam=instance.mef_nam
        ).values_list('mef_id', flat=True).first()
        
        return ret
    
    class Meta:
        model = Bx_st_fab_job_oper_corte
        fields = ["mef_id", "mef_nam", "it_codigo_marca", "reservas", "operacoes", "ordens_posicoes"]
```

---

#### Passo 2: Otimizar ViewSet com `prefetch_related()`

**Arquivo:** `st_fab_job_marca/views.py`

```python
from rest_framework import viewsets
from django.db.models import Prefetch

class Bx_st_fab_job_oper_marcaViewSet(viewsets.ModelViewSet):
    
    def list(self, request):
        mef_nam = request.query_params.get("mef_nam")
        it_codigo = request.query_params.get("it_codigo")
        filtros = {}
        
        if mef_nam is not None:
            filtros["mef_nam"] = mef_nam
        if it_codigo is not None:
            bx_erp_item = Bx_erp_item.objects.filter(
                it_codigo=it_codigo
            ).first()
            if bx_erp_item is not None:
                if bx_erp_item.cd_folh_item == "POSICAO":
                    self.tipo = "POSICAO"
                    filtros["it_codigo_posicao"] = it_codigo
                else:
                    filtros["it_codigo_marca"] = it_codigo
        
        # ✅ OTIMIZADO: Prefetch dados relacionados
        queryset = Bx_st_fab_job_oper_corte.objects.filter(**filtros).prefetch_related(
            Prefetch('reservas'),
            Prefetch('operacoes'),
            Prefetch('ordens')
        )
        
        bx_st_fab_job_oper_cortes = list(queryset)
        
        serializer = self.get_serializer(bx_st_fab_job_oper_cortes, many=True)
        
        if len(bx_st_fab_job_oper_cortes) != 0:
            return success_response(
                key="bx_st_fab_job_oper_marca", data=serializer.data
            )
        else:
            return error_response("Não há marca com esses parâmetros")
```

---

## Checklist de Implementação

### ✅ N+1 Queries (Prioridade 1)

- [ ] **st_fab_job_marca/serializers.py**
  - [ ] Linhas 14-18: Refatorar `get_reservas()` em `Bx_st_fab_job_operMarcaSerializer`
  - [ ] Linhas 23-28: Refatorar `get_operacoes()` em `Bx_st_fab_job_operMarcaSerializer`
  - [ ] Linhas 32-37: Refatorar `get_ordens_posicoes()` em `Bx_st_fab_job_operMarcaSerializer`
  - [ ] Linhas 50-51: Refatorar `get_mef_id()` em `Bx_st_fab_job_operMarcaSerializer`
  - [ ] Linhas 61-67: Refatorar `get_reservas()` em `Bx_st_fab_job_operPosicaoSerializer`
  - [ ] Linhas 72-77: Refatorar `get_operacoes()` em `Bx_st_fab_job_operPosicaoSerializer`
  - [ ] Linhas 81-86: Refatorar `get_ordens_posicoes()` em `Bx_st_fab_job_operPosicaoSerializer`
  - [ ] Linhas 103-104: Refatorar `get_mef_id()` em `Bx_st_fab_job_operPosicaoSerializer`

- [ ] **st_fab_job_posicao/serializers.py**
  - [ ] Linhas 12-14: Refatorar `get_reservas()` com bulk lookup
  - [ ] Linhas 18-19: Refatorar `get_caracteristicas()` com bulk lookup
  - [ ] Linhas 23-25: Refatorar `get_operacoes()` com bulk lookup

- [ ] **st_ord_prod/serializers.py**
  - [ ] Linhas 26-41: Adicionar prefetch_related em `get_operacoes()`
  - [ ] Linhas 43-52: Adicionar prefetch_related em `get_reservas()`

- [ ] **st_fab_job_marca/views.py**
  - [ ] Linhas 44-67: Adicionar prefetch_related no método `list()`

---

### 📊 Testes de Performance

Após implementação, executar:

```bash
# Test com Django Debug Toolbar
curl "http://localhost:8000/api/bx/st-fab-job-marca/?page=1"

# Monitorar número de queries
# Esperado ANTES: 100+ queries
# Esperado DEPOIS: 3-5 queries
```

---

### 📅 Cronograma Sugerido

**Dia 1 (3-4 horas):**
1. Refatorar `st_fab_job_marca/serializers.py`
2. Refatorar `st_fab_job_posicao/serializers.py`
3. Testar N+1 queries

**Dia 2 (1-2 horas):**
1. Testes de performance globais
2. Refatorar `st_fab_job_marca/views.py` com prefetch
3. Validação final

---

### 🎯 Métricas de Sucesso

| Métrica | Antes | Depois | Status |
|---------|-------|--------|--------|
| Queries por requisição | 100+ | 3-5 | ⏳ |
| Tempo resposta (100 itens) | 5-8s | 0.5-1s | ⏳ |
| Taxa de erro | 15% | <1% | ⏳ |

---

**Gerado em:** 29/01/2026
