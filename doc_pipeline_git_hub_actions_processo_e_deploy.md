# Deploy Automático — Guia Rápido

Como publicar novas versões do projeto nos servidores.

---

## Como funciona?

Quando você quiser publicar uma nova versão:

1. **Crie uma tag** com o nome do ambiente
2. **Envie a tag** para o GitHub
3. **Pronto!** O sistema faz o resto automaticamente

```
Você cria a tag → GitHub detecta → Deploy acontece sozinho
```

---

## Qual tag usar?

| Ambiente | Tag | Exemplo | Servidor |
|----------|-----|---------|----------|
| **Desenvolvimento** | `dev_` + versão | `dev_1.0.0` | `C:\apps\projeto\dev` |
| **Homologação** | `hml_` + versão | `hml_1.0.0` | `C:\apps\projeto\hml` |
| **Produção** | `prod_` + versão | `prod_1.0.0` | `C:\apps\projeto\prod` |

---

## Passo a passo

### Publicar em DEV
```bash
git tag dev_1.0.0
git push origin dev_1.0.0
```

### Publicar em Homologação
```bash
git tag hml_1.0.0
git push origin hml_1.0.0
```

### Publicar em Produção
```bash
git tag prod_1.0.0
git push origin prod_1.0.0
```

> 💡 **Dica:** Use números de versão que façam sentido (ex: `prod_2.3.1`).

---

## O que acontece por trás?

```
┌─────────────────────────────────────────────────────────┐
│  1. GitHub recebe a tag                                 │
│  2. Pipeline conecta nos servidores (via SSH seguro)    │
│  3. Código é atualizado no servidor correto             │
│  4. Aplicação é reiniciada automaticamente              │
└─────────────────────────────────────────────────────────┘
```

A conexão passa por 3 servidores em cadeia (por segurança):
```
Internet → Servidor 1 → Servidor 2 → Servidor Final
```

---

## Por que usar tags?

| Benefício | Explicação |
|-----------|------------|
| ✅ **Sem acidentes** | Commit não faz deploy — só a tag faz |
| ✅ **Rastreável** | Cada versão tem um nome claro |
| ✅ **Fácil voltar atrás** | Basta fazer deploy de uma tag antiga |
| ✅ **Ambiente certo** | O prefixo da tag define onde vai (dev/hml/prod) |

---

## Requisitos (para a equipe de infra)

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
📦 Quer publicar? → Crie uma tag → Envie pro GitHub → Pronto!
```

| Ação | Comando |
|------|---------|
| Deploy DEV | `git tag dev_X.X.X && git push origin dev_X.X.X` |
| Deploy HML | `git tag hml_X.X.X && git push origin hml_X.X.X` |
| Deploy PROD | `git tag prod_X.X.X && git push origin prod_X.X.X` |

---

> 📝 O arquivo de configuração completo está em `.github/workflows/deploy-por-tag.yml`

