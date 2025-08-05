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

<img width="587" height="72" alt="image" src="https://github.com/user-attachments/assets/ac36a808-0f4c-44d1-a265-c3dc03b1d621" />

Les Erreur possibles:
- Flux fermé entre ArgoCD et ADFS:   vous avez un timeout
  Failed to query provider "https://adfs.grt.vva/adfs": Get "https://adfs.grt.vva/adfs/.well-known/openid-configuration": dial tcp 10.2D.1o.0:443: i/o timeout
- Problème de certificat: Rajouter la CA de votre entité a la suite de la configuration de l'OIDC

# Pour la Gestion des permissions:

