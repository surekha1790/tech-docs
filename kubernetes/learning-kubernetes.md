## Learning Kubernetes

Kubernetes is used to manage docker containers. Auto start/stop and scale pods.
```shell
  kubectl apply -f deployment.yaml -- to apply changes from yaml file
  kubectl get pods  -- list all available pods
  kubectl describe pod <pod-name>  -- get details
  kubectl logs <pod-name>
  
  kubectl rollout status deployment web-deployment
  kubectl rollout history deployment web-deployment
  kubectl rollout undo deployment web-deployment  
  
  kubectl get services
  
  -- when config is changed, run below to get the updated value
  kubectl rollout restart deployment <name>
```