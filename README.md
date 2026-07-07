# Casino Backend

Backend del casino online, construido con Node.js y Express.

## Construir (Build)
Para instalar las dependencias y preparar el entorno:
```bash
npm install
```

## Probar (Test)
Para ejecutar las pruebas unitarias (Jest):
```bash
npm test
```

## Desplegar (Deploy)
El despliegue está automatizado mediante GitHub Actions. Al hacer un push a la rama `deploy`, se ejecutará el flujo CI/CD:
1. Ejecución de tests.
2. Construcción de la imagen Docker.
3. Subida a Amazon ECR.
4. Despliegue en el clúster Amazon EKS.

Para aplicar manualmente en Kubernetes:
```bash
kubectl apply -f ../k8s/backend.yaml
```

## Troubleshooting
- **Pod en CrashLoopBackOff**: Revisa los logs del contenedor con `kubectl logs deployment/casino-backend`.
- **Problemas de conexión a BD**: Verifica que los secretos de Postgres existan (`kubectl get secret casino-secrets`).
