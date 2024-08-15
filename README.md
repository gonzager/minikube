# Deployar Aplicacion en Minikube

Este es un proyecto instructivo que te ayudará a lograr desplegar tu aplicación en un cluster de kuberntes local. El cluster local que vamos a utilizar en **minikube** y lo podes descargar desde esta página oficial. [Download](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download)

Para poder hablar con el cluster vamos a necesitar que tambien tengas otra herramienta instalada, en este caso en [kubectl](https://kubernetes.io/docs/tasks/tools/) 

Lo más probable es que cuando hayas instalado minikube ya te deje todo configurado todo el cluster y ademos te puedas comunicar con él a través de kubectl sin necesidad de hacer ninguna configuración extra.

Para arrancar el minikube solo tenes que correr el comando
```
minikube start
```

Para probar que podes comunicarte con minikube podes alguno de los siguientes comandos
```
kubectl get pods -o wide
```

Para lanzar el dashbord de minikube donde vas a poder visualizar y monitoreas todo el cluster podes correr el comando 
```
minikube dashboard
````


## Enviar Imagen a Mimikube
Para que el build de una imagen Docker se realice dentro del entorno de Minikube, puedes utilizar el siguiente comando:
```
eval $(minikube docker-env)
```
Este comando configura tu terminal para que el cliente de Docker se conecte al demonio de Docker que está corriendo dentro de Minikube. De esta manera, las imágenes Docker que construyas se crearán dentro de Minikube, lo que facilita su uso en pods y despliegues sin necesidad de empujar la imagen a un registro externo. .

Después de ejecutar este comando, puedes construir la imagen Docker como lo harías normalmente:

```
docker build -t mi-imagen:tag .
```
Recuerda que esta configuración solo afecta a la sesión actual de la terminal. Si abres una nueva terminal, tendrás que ejecutar el comando eval $(minikube docker-env) nuevamente.


## Conectarme al pod de minikube
Es posible que el algun momento necesites conectarte al pod de minikube para crear directorios, verificar imagenes que tiene, etc. Podes ingresar al minikube ejecutando el siguiente comando

```
minikube ssh
```

## Habilitar Ingress
Si ya tienes Minikube corriendo y no estás seguro si el complemento de Ingress está habilitado, puedes verificarlo con:
```
minikube addons list
```
Si ingress no está habilitado, puedes habilitarlo con el siguiente comando:
```
minikube addons enable ingress
```

Una vez habilitado, puedes comprobar que el Ingress Controller está en ejecución con:
```
kubectl get pods -n kube-system
```