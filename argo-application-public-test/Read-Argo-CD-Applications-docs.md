# Argo CD Applications de Teste (Namespace Compartilhado)

**Data:** 2026-02-26
**Ambiente:** Homolog
**Cluster:** EKS homolog
**Namespace alvo:** `homolog`
**Projeto Argo CD:** `homolog`

## Objetivo

Documentar aplicações de teste criadas no Argo CD para validar o comportamento de:

* múltiplas aplicações no mesmo namespace compartilhado
* múltiplas aplicações no mesmo `AppProject`
* remoção de uma aplicação sem impactar as demais
* funcionamento de `prune: true` em namespace compartilhado
* comportamento de `CreateNamespace=true`

---

## Contexto

Foi validado que é possível ter **mais de uma aplicação Argo CD** no mesmo:

* namespace Kubernetes (`homolog`)
* projeto Argo CD (`homolog`)

sem que uma aplicação apague automaticamente a outra.

### Regra prática observada

O Argo CD separa aplicações pelo conjunto de recursos gerenciados por cada `Application`, com base em:

* `repoURL`
* `targetRevision`
* `path`

Ou seja, duas apps podem compartilhar o mesmo namespace, desde que **não tentem gerenciar o mesmo recurso com o mesmo nome**.

---

## Aplicação 1 — `guestbook-test`

### Finalidade

Aplicação de teste simples usando o repositório oficial de exemplos do Argo CD.

### Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-test
  namespace: argocd
spec:
  project: homolog
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: master
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: homolog
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Observações

* Usa namespace compartilhado: `homolog`
* Usa projeto: `homolog`
* `prune: true` remove apenas os recursos dessa aplicação que saírem do Git
* `CreateNamespace=true` apenas cria o namespace se ele não existir

---

## Aplicação 2 — `guestbook-helm-test`

### Finalidade

Segunda aplicação de teste no mesmo namespace compartilhado para validar convivência entre apps.

### Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-helm-test
  namespace: argocd
spec:
  project: homolog
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: master
    path: helm-guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: homolog
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Observações

* Também usa o namespace `homolog`
* Também usa o projeto `homolog`
* Serve para validar que duas apps diferentes podem coexistir no mesmo namespace
* Ao remover uma, a outra deve permanecer intacta

---

## Resultado Esperado do Teste

Após aplicar as duas aplicações:

1. ambas devem aparecer no Argo CD
2. ambas devem sincronizar normalmente
3. os recursos de ambas devem coexistir no namespace `homolog`
4. ao deletar uma aplicação:

   * a outra continua funcionando
   * o namespace `homolog` continua existindo
   * apenas os recursos da app removida devem ser apagados

---

## O que NÃO deve acontecer

Excluir uma aplicação **não deve**:

* apagar outra aplicação no mesmo namespace
* apagar o namespace `homolog`
* remover recursos de outra app

### Exceção / Risco

Pode haver problema se duas apps tentarem gerenciar o **mesmo recurso**, por exemplo:

* mesmo `kind`
* mesmo `name`
* mesmo `namespace`

Exemplo de conflito:

* App A cria `Service/app-x` em `homolog`
* App B também tenta criar `Service/app-x` em `homolog`

Nesse caso, pode haver:

* conflito de ownership
* sobrescrita
* comportamento inesperado no delete/sync

---

## Recomendação de Boas Práticas

### Para namespace compartilhado

Pode usar normalmente, desde que:

* cada app tenha nomes únicos para seus recursos
* uma app não tente gerenciar recursos da outra
* evite compartilhar `ConfigMap`, `Secret` e `Service` sem necessidade

### Para namespace dedicado por app

Quando o namespace for exclusivo da aplicação, o cenário fica ainda mais previsível.

---

## Sobre `prune: true`

Neste teste foi mantido:

```yaml
prune: true
```

### Motivo

Esse comportamento mantém o cluster alinhado com o Git, removendo apenas recursos da **própria aplicação** que saíram do repositório.

### Importante

`prune: true` **não é o que apaga namespace compartilhado**.

O namespace só vira risco se ele estiver sendo gerenciado explicitamente como recurso da própria app, por exemplo:

```yaml
kind: Namespace
metadata:
  name: homolog
```

Se o namespace **não** estiver versionado dentro da app, o `CreateNamespace=true` sozinho **não faz o Argo CD apagar o namespace depois**.

---

## Fonte dos manifests usados

### Repositório oficial de exemplos do Argo CD

* Repositório: `argoproj/argocd-example-apps`
* URL: `https://github.com/argoproj/argocd-example-apps`

### Paths usados

* `guestbook`
* `helm-guestbook`

### Finalidade

Usados apenas para laboratório e validação de comportamento do Argo CD.

> Recomendação: usar esse repositório apenas para testes, nunca como padrão de homologação/produção.

---

## Fonte da orientação técnica

Esta documentação foi montada com base em:

* testes realizados no ambiente `homolog`
* comportamento observado diretamente no Argo CD
* boas práticas de GitOps discutidas durante a configuração do ambiente

---

## Assistência de IA utilizada

**Ferramenta:** ChatGPT
**Modelo utilizado:** GPT-5.2 Thinking
**Tipo de IA:** modelo de linguagem (LLM, Large Language Model) com suporte a raciocínio e assistência técnica em infraestrutura, Kubernetes e GitOps.

### Como a IA ajudou

A IA foi utilizada para:

* revisar manifests de `Application`
* validar comportamento de `prune`, `selfHeal` e `CreateNamespace`
* explicar o impacto de múltiplas apps no mesmo namespace
* ajudar a documentar a configuração de forma clara e operacional

### Observação

As decisões finais de implantação, segurança e operação devem sempre ser validadas no cluster e no contexto real do ambiente.

---

## Comandos úteis para validação

### Ver aplicações

```bash
kubectl get applications -n argocd
```

### Ver recursos no namespace compartilhado

```bash
kubectl get all -n homolog
```

### Ver detalhes de uma aplicação

```bash
kubectl get application guestbook-test -n argocd -o yaml
kubectl get application guestbook-helm-test -n argocd -o yaml
```

---

## Conclusão

O teste demonstra que:

* é seguro ter múltiplas aplicações no mesmo namespace compartilhado
* é seguro ter múltiplas aplicações no mesmo `project: homolog`
* remover uma aplicação **não deve apagar as outras**
* o namespace compartilhado **não deve ser removido** apenas por deletar uma app
* o cuidado principal está em evitar conflito de nomes entre recursos

---

Se quiser, eu também posso te entregar uma **segunda página `.md`** só com:

* **boas práticas de segurança para Argo CD**
* **padrão de nomes para evitar conflito entre apps**
* **modelo de AppProject para homolog / produção**
