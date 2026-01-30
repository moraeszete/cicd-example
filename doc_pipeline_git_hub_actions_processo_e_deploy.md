# Deploy Automático — Guia Rápido

Como publicar novas versões do projeto nos servidores.

---

## Como funciona?

Quando você quiser publicar uma nova versão:

1. **Confirme branch atual** para que seja possivel tracking das tags
2. **Crie uma tag** com o nome do ambiente que deseja atualizar
3. **Envie a tag** para o GitHub
4. **Pronto!** O Actions faz o resto.

---

## Qual tag usar?

| Ambiente | Tag | Exemplo | Path Final |
|----------|-----|---------|----------|
| **Desenvolvimento** | `dev_` + versão | `dev_1.0.0` | `C:\Buildx\Buildx_dev` |
| **Homologação** | `hml_` + versão | `hml_1.0.0` | `C:\Buildx\Buildx_hml` |
| **Produção** | `prod_` + versão | `prod_1.0.0` | `C:\Buildx\Buildx_prod` |

---

## Passo a passo

### Publicar em DEV | HML | PROD
```bash
git branch -v; 
git tag -a dev_1.0.0 -m "comentario pertinente";
git push origin dev_1.0.0;
(ou git push origin --tags)
```

> **Dica:** Use números de versão que façam sentido (ex: `prod_1.0.9` -> `prod_1.1.0` ).

---

## Como Logica do pipeline funciona?

```
┌─────────────────────────────────────────────────────────┐
│  1. GitHub recebe a tag (setando o path final)          │
│  2. Pipeline conecta nos servidores (via SSH seguro)    │
│  3. Código é atualizado no servidor final               │
│  4. Aplicação é reiniciada automaticamente              │
└─────────────────────────────────────────────────────────┘
```

A conexão passa pelos 3 servidores em cadeia:
```
 Internet (Git Actions)→ SINOSBYTE → 201.44.172.227 → uma007s.metasa.com.br
```
> Setar as chaves de ssh com a mesma senha (senha opcional)

---

## Por que usar tags?

| Benefício | Explicação |
|-----------|------------|
| ✅ **Sem acidentes** | Commit não faz deploy — só a tag faz |
| ✅ **Rastreável** | Cada versão tem um nome claro |
| ✅ **Fácil voltar atrás** | Basta fazer deploy de uma tag antiga |
| ✅ **Ambiente certo** | O prefixo da tag define onde vai (dev/hml/prod) |

---

## Requisitos (infra)

- OpenSSH ativo nos Windows Servers
- Chaves SSH configuradas no GitHub (Secrets)
- `make` instalado no servidor final

---

## Secrets necessários no GitHub

| Secret | Descrição |
|--------|-----------|
| `SSH_KEY_SBYTE` | Chave do servidor de entrada |
| `SSH_KEY_METASA_1` | Chave do servidor intermediário |
| `SSH_KEY_METASA_2` | Chave do servidor final |
| `SSH_PASSPHRASE` | Senha das chaves (se houver) |
| `SSH_USER_SBYTE` | Usuário do servidor de entrada |
| `SSH_USER_METASA` | Usuário dos servidores Metasa |
| `HOST_SBYTE` | IP/hostname do servidor de entrada |
| `HOST_METASA_1` | IP/hostname do servidor intermediário |
| `HOST_METASA_2` | IP/hostname do servidor final |

---

## Resumo

```
Quer publicar? → Crie uma tag → Envie pro GitHub → Pronto!
```

| Ação | Comando |
|------|---------|
| Deploy DEV | `git tag dev_X.X.X && git push origin dev_X.X.X` |
| Deploy HML | `git tag hml_X.X.X && git push origin hml_X.X.X` |
| Deploy PROD | `git tag prod_X.X.X && git push origin prod_X.X.X` |

---