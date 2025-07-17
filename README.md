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

---
🔐 Étape 1 : Activer mTLS dans Istio (Authentification)
C'est ce que tu fais déjà, mais pour être complet :

yaml
Copier
Modifier
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-namespace
spec:
  mtls:
    mode: STRICT
Cela oblige tous les pods dans ce namespace à utiliser mTLS. Cela assure l’identité du client (authentification mutuelle entre proxies).

🪪 Étape 2 : Keycloak émet des JWT (OAuth2 / OIDC)
Crée un realm dans Keycloak.

Crée un client (confidential ou public).

Les clients de ton mesh (ex : les frontends, ou API-gateway) doivent s’authentifier auprès de Keycloak pour obtenir un JWT.

Ce JWT est ensuite envoyé avec chaque requête HTTP, dans le header Authorization: Bearer <token>.

🛡️ Étape 3 : Istio valide les JWT (et fait l'autorisation)
Tu vas utiliser les ressources RequestAuthentication et AuthorizationPolicy.

1. RequestAuthentication : pour dire à Istio comment vérifier les JWT
yaml
Copier
Modifier
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-keycloak
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: my-api
  jwtRules:
    - issuer: "https://keycloak.example.com/realms/myrealm"
      jwksUri: "https://keycloak.example.com/realms/myrealm/protocol/openid-connect/certs"
2. AuthorizationPolicy : pour contrôler l’accès selon les claims
Exemple : autoriser uniquement les utilisateurs avec le rôle admin dans le realm.

yaml
Copier
Modifier
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-admins
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: my-api
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["https://keycloak.example.com/realms/myrealm/*"]
      when:
        - key: request.auth.claims[realm_access].roles
          values: ["admin"]
🧠 Résumé
Cible	Solution
Authentification mutuelle	Istio + mTLS (identité de service)
Authentification utilisateur	JWT signé par Keycloak
Autorisation	AuthorizationPolicy basée sur les claims JWT
Avantage	Tu n’as pas besoin d’ajouter de lib auth dans tes apps
