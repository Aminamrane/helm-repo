# Helm Repository

Repository contenant les charts Helm pour déployer les applications sur Kubernetes.

## Structure

```
helm-repo/
├── platform/          # Umbrella chart (chart principal)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── auth/              # Subchart pour le service auth
├── users/             # Subchart pour le service users
├── items/             # Subchart pour le service items
├── frontend/          # Subchart pour le service frontend
├── Jenkinsfile        # Pipeline CI/CD Jenkins
└── README.md
```

## Pipeline Jenkins

La pipeline Jenkins effectue les étapes suivantes :

1. **Checkout** : Récupération des charts Helm
2. **Validate Helm Charts** : Validation de la syntaxe des charts
3. **Update Dependencies** : Mise à jour des dépendances Helm
4. **Update Image Versions** : Mise à jour des versions d'images Docker dans values.yaml
5. **Deploy to Kubernetes** : Déploiement sur le cluster Kubernetes
6. **Verify Deployment** : Vérification du déploiement

## Paramètres de la pipeline

- **SERVICE** : Service à déployer (all, backend, frontend, auth, users, items, gateway)
- **IMAGE_VERSION** : Version/tag de l'image Docker à déployer
- **NAMESPACE** : Namespace Kubernetes (dev/prod)
- **ENVIRONMENT** : Environnement (dev/prod)

## Utilisation

### Déploiement complet
```groovy
build job: 'helm-deploy', 
    parameters: [
        string(name: 'SERVICE', value: 'all'),
        string(name: 'IMAGE_VERSION', value: '123'),
        string(name: 'NAMESPACE', value: 'dev')
    ]
```

### Déploiement d'un service spécifique
```groovy
build job: 'helm-deploy', 
    parameters: [
        string(name: 'SERVICE', value: 'frontend'),
        string(name: 'IMAGE_VERSION', value: '456'),
        string(name: 'NAMESPACE', value: 'dev')
    ]
```

## Configuration Jenkins

Créer les credentials suivants dans Jenkins :
- **kubeconfig-credentials** : Fichier kubeconfig pour accéder au cluster Kubernetes

## Images Docker

Les images sont référencées avec le préfixe `leogrv22/` :
- `leogrv22/auth`
- `leogrv22/users`
- `leogrv22/items`
- `leogrv22/gateway`
- `leogrv22/frontend`

## Appels depuis d'autres pipelines

Cette pipeline peut être appelée depuis :
- **Backend pipeline** : Après le push des images backend
- **Frontend pipeline** : Après le push de l'image frontend

