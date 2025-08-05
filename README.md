# SSO-ADFS-argoCD
# 🔐 Configuration du SSO Argo CD avec ADFS (via OIDC)

Ce guide explique comment configurer l’authentification Single Sign-On (SSO) dans **Argo CD** en utilisant **ADFS** (Active Directory Federation Services) comme **fournisseur OIDC**.

---

## 🧩 Prérequis

- Un cluster Kubernetes avec Argo CD déployé.
- Accès administrateur à **ADFS**.
- Un **nom de domaine public** (ou interne) pour accéder à Argo CD.
- ADFS accessible via HTTPS avec un certificat valide.

---

## 1. ⚙️ Configuration ADFS (Identity Provider)

### ➤ Créer une application dans ADFS

1. Ouvrir la **console ADFS**.
2. Aller dans **Application Groups** → _Add Application Group_.
3. Donner un nom, ex: `ArgoCD`, puis choisir :
   - **Server application accessing a web API**.
4. Ajouter une **application client** :
   - Client Identifier : `argocd`
   - Redirect URI : `https://ARGOCD_URL/api/dex/callback`
5. Générer et noter le **Client Secret**.
6. Ajouter les **scopes** autorisés :
   - `openid`, `profile`, `email`, `groups` (si nécessaire).

---

## 2. 🔧 Configuration d’Argo CD

### ➤ Modifier le ConfigMap `argocd-cm`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://ARGOCD_URL
  oidc.config: |
    name: ADFS
    issuer: https://ADFS_DOMAIN/adfs
    clientID: argocd
    clientSecret: <CLIENT_SECRET>
    requestedScopes: ["openid", "profile", "email", "groups"]



3. 🔐 Configuration RBAC (contrôle d’accès)
➤ Modifier argocd-rbac-cm
yaml
Copier
Modifier
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    g:DOMAIN\\GroupName, role:admin
Remplacez DOMAIN\\GroupName par le groupe Active Directory autorisé à accéder à Argo CD en tant qu’administrateur.

4. 🔄 Appliquer et redémarrer Argo CD
bash
Copier
Modifier
kubectl -n argocd apply -f argocd-cm.yaml
kubectl -n argocd apply -f argocd-rbac-cm.yaml
kubectl -n argocd rollout restart deployment argocd-server


5. ✅ Vérification
Accéder à : https://ARGOCD_URL

Cliquer sur "Login via ADFS"

S’authentifier avec ADFS

Redirection vers Argo CD avec accès selon le groupe configuré

🧪 Test & Dépannage

Problème	Solution
Redirection ADFS ne fonctionne pas	Vérifiez les URI de redirection dans ADFS
Aucune réponse de ADFS	Testez https://ADFS_DOMAIN/adfs/.well-known/openid-configuration
Erreur d’authentification	Vérifiez clientID et clientSecret
Pas de bouton de login ADFS	Vérifiez le oidc.config dans argocd-cm



Les Erreur possibles:
- Flux fermé entre ArgoCD et ADFS:   vous avez un timeout
  Failed to query provider "https://adfs.grt.vva/adfs": Get "https://adfs.grt.vva/adfs/.well-known/openid-configuration": dial tcp 10.2D.1o.0:443: i/o timeout
- Problème de certificat: Rajouter la CA de votre entité a la suite de la configuration de l'OIDC
<img width="587" height="115" alt="image" src="https://github.com/user-attachments/assets/f6dc0d30-b322-4dba-b6ff-7fbe2f8f4604" />

# Pour la Gestion des permissions: modifier la configmap  argocd-rbac-cm

# 📄 Explication du fichier `policy.csv` pour Argo CD RBAC

Ce document explique en détail la configuration suivante de `policy.csv` utilisée pour restreindre les permissions d’un utilisateur dans Argo CD.

---

## 🔐 Contexte

Dans Argo CD, la **configMap `argocd-rbac-cm`** permet de définir des **politiques d’accès** pour les utilisateurs (authentifiés via SSO comme ADFS). Ces règles définissent **qui** peut faire **quoi**, **sur quoi**, et **où**.

La syntaxe d'une ligne dans `policy.csv` est :

```
p, sujet, action, objet, ressource, effet
```

---

## 🧾 Analyse ligne par ligne

```yaml
p, user:john.doe@example.com, applications, get, XX/*, allow
```
- ✅ Autorise l'utilisateur `john.doe@example.com` à **lire les applications** dans le namespace logique (Argo CD app) `XX`.

---

```yaml
p, user:john.doe@example.com, applications, create, XX/*, allow
```
- ✅ Autorise la **création d'applications Argo CD** dans le namespace `XX`.

---

```yaml
p, user:john.doe@example.com, applications, update, XX/*, allow
```
- ✅ Autorise la **modification des applications** existantes dans le namespace `XX`.

---

```yaml
p, user:john.doe@example.com, applications, delete, XX/*, allow
```
- ✅ Autorise la **suppression d'applications** dans le namespace `XX`.

---

```yaml
p, user:john.doe@example.com, clusters, get, *, allow
```
- ✅ Autorise l'accès **en lecture seule aux clusters** connectés à Argo CD.

---

```yaml
p, user:john.doe@example.com, projects, get, *, allow
```
- ✅ Autorise la lecture des **projets Argo CD**.

---

## ✅ Résumé des permissions

| Action     | Ressource     | Portée                 |
|------------|----------------|-------------------------|
| Lire       | Applications    | Namespace `XX`         |
| Créer      | Applications    | Namespace `XX`         |
| Modifier   | Applications    | Namespace `XX`         |
| Supprimer  | Applications    | Namespace `XX`         |
| Lire       | Clusters        | Tous                   |
| Lire       | Projets         | Tous                   |

---

## 🔒 Limitations

- L'utilisateur **n’a pas accès aux autres namespaces** que `XX`.
- Il **ne peut pas gérer** les clusters ni les projets (seulement lecture).
- Il **peut uniquement gérer des applications** dans `XX`.

---

## 🛠 Suggestions supplémentaires

- Utiliser un **groupe AD** au lieu d’un utilisateur pour une gestion centralisée.
- Créer un **rôle nommé** et y associer des groupes avec la directive `g, group, role:nom`.
- Étendre les permissions si nécessaire (par ex. accès aux logs via RBAC Kubernetes).

data:
  policy.csv: |
    p, user:john.doe@example.com, applications, get, XX/*, allow
    p, user:john.doe@example.com, applications, create, XX/*, allow
    p, user:john.doe@example.com, applications, update, XX/*, allow
    p, user:john.doe@example.com, applications, delete, XX/*, allow
    p, user:john.doe@example.com, clusters, get, *, allow
    p, user:john.doe@example.com, projects, get, *, allow

<img width="495" height="187" alt="image" src="https://github.com/user-attachments/assets/68a16b1c-534b-4070-aa08-3ebece41d0a6" />

# Projet admin
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  # Rôle par défaut : lecture seule
  policy.default: role:readonly

  # Définition des rôles et permissions
  policy.csv: |
    # Rôle personnalisé pour la gestion des projets et applications
    p, role:project-admin, applications, *, *, allow
    p, role:project-admin, projects, *, *, allow
    p, role:project-admin, clusters, get, *, allow
    p, role:project-admin, repositories, get, *, allow
    p, role:project-admin, repositories, list, *, allow
    p, role:project-admin, logs, get, *, allow
    p, role:project-admin, exec, create, *, allow

    # Associer un utilisateur au rôle
    g, john.doe@example.com, role:project-admin


# Full Admin

-----
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  # Rôle par défaut (lecture seule pour les autres utilisateurs)
  policy.default: role:readonly

  # Définition des règles RBAC
  policy.csv: |
    # Rôle full admin
    p, role:custom-admin, *, *, *, allow

    # Associer un utilisateur au rôle
    g, john.doe@example.com, role:custom-admin

# Limitation a un cluster
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  # Rôle par défaut : lecture seule
  policy.default: role:readonly

  policy.csv: |
    # Définir le rôle "deployer-kube1" avec droits limités
    p, role:deployer-kube1, applications, get, *, allow
    p, role:deployer-kube1, applications, create, *, allow
    p, role:deployer-kube1, applications, update, *, allow
    p, role:deployer-kube1, applications, delete, *, allow
    p, role:deployer-kube1, applications, sync, *, allow
    p, role:deployer-kube1, applications, override, *, allow
    p, role:deployer-kube1, clusters, get, https://10.0.0.1:6443, allow
    p, role:deployer-kube1, repositories, get, *, allow
    p, role:deployer-kube1, logs, get, *, allow

    # Limiter la création aux applications déployées sur le cluster kube1
    p, role:deployer-kube1, applications, create, *, allow
    p, role:deployer-kube1, applications, sync, *, allow
    p, role:deployer-kube1, applications, override, *, allow
    p, role:deployer-kube1, applications, update, *, allow

    # Restreindre aux applications qui ciblent ce cluster
    p, role:deployer-kube1, applications, *, *, allow, obj.cluster == "https://10.0.0.1:6443"

    # Associer un utilisateur à ce rôle
    g, jane.doe@example.com, role:deployer-kube1

#  Si on a des users dans un groupe Dev au niveau de ADFS, on peut restreindre les droits a ces users

apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly

  policy.csv: |
    # Rôle dev limité (pas d'accès aux projets)
    p, role:dev-limited, applications, get, *, allow
    p, role:dev-limited, applications, list, *, allow
    p, role:dev-limited, applications, create, *, allow
    p, role:dev-limited, applications, sync, *, allow
    p, role:dev-limited, logs, get, *, allow

    # Ne PAS mettre de règle sur "projects", donc pas de droit de créer
    # Associer le groupe ADFS "dev" à ce rôle
    g, dev, role:dev-limited
