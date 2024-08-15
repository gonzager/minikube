# Generación de un Certificado Autofirmado y usarlo en un MiniKube

## Generar un archivo que los datos del certificado

El archivo proporcionado [openssl-san.conf](./openssl-san.cnf) es una configuración para OpenSSL que permite generar certificados SSL/TLS personalizados. Dentro de este archivo, se especifican diversos parámetros que determinan las propiedades del certificado que se generará.

### ¿Qué es SAN?
SAN significa Subject Alternative Name (Nombre Alternativo del Sujeto). Es una extensión en los certificados X.509 que permite especificar múltiples nombres de dominio (o direcciones IP) para los cuales el certificado es válido. Esto es especialmente útil cuando deseas que un único certificado sea válido para varios dominios o subdominios.

Por ejemplo, un certificado con un SAN que incluya example.com y www.example.com será válido para ambos dominios. Además, los wildcards (comodines) pueden usarse para cubrir múltiples subdominios, como *.example.com, que sería válido para sub.example.com, another.sub.example.com, etc.

### Desglose del Archivo de Configuración

```
[ req ]
default_bits        = 2048
distinguished_name  = req_distinguished_name
x509_extensions     = v3_req
prompt              = no

[ req_distinguished_name ]
countryName                  = AR
stateOrProvinceName          = Buenos Aires
localityName                 = La Plata
organizationName             = flux It
organizationalUnitName       = Technology
commonName                   = flux-bancogalicia.com

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = flux-bancogalicia.com
DNS.2 = *.flux-bancogalicia.com
```

### Secciones y Parámetros:

- [ req ]
    - default_bits = 2048: Establece el tamaño de la clave a 2048 bits.
    - distinguished_name = req_distinguished_name: Especifica la sección que contiene los nombres distinguidos.
    - x509_extensions = v3_req: Apunta a la sección que contiene las extensiones X.509 v3.
    - prompt = no: Evita que OpenSSL solicite información durante la generación del certificado; usa los valores proporcionados.

- [ req_distinguished_name ]
    - Contiene la información del sujeto del certificado, como país, estado, localidad, organización, etc.
    - commonName = flux-bancogalicia.com: Especifica el nombre sin wildcard. Es la forma tradicional de especificar para qué dominio es válido un certificado. Aunque puedes usar un wildcard en el CN, la práctica moderna recomienda poner el dominio principal en el CN y manejar los wildcards en los SANs.

- [ v3_req ]
    - subjectAltName = @alt_names: Indica que la sección alt_names contiene los SANs.

- [ alt_names ]
    -   DNS.1 = flux-bancogalicia.com: Agrega un SAN para el dominio raíz.
    -   DNS.2 = *.flux-bancogalicia.com: Agrega un SAN para todos los subdominios de flux-bancogalicia.com.

### Importancia de SAN en Certificados
Incluir SANs en un certificado es crucial para garantizar que los navegadores y otros clientes reconozcan y confíen en el certificado para los dominios especificados. Sin SANs, es posible que los navegadores modernos muestren errores de seguridad, incluso si el campo Common Name coincide con el dominio.

## Pasos para Generar el Certificado

### Paso 1 - Archivo con los datos del certificado
Guarda el contenido anterior en un archivo, por ejemplo, [openssl-san.cnf](./openssl-san.cnf).


### Paso 2 - Generar el Certificado y Clave Privada
Generar el certificado con el siguiente comando ssl
```
openssl req -new -x509 -nodes -out server.crt -keyout server.key -days 365 -config openssl-san.cnf
```
Este comando genera un certificado autofirmado válido por 365 días con los SANs especificados.

Se generarán 2 archivos como resultado de ejecucion del comando que seran
- server.crt (-out server.crt o el nombre que se especifique después del -out) que contiene el certificado
- server.key (-keyout server.key o el nombre que se especifique después del --keyout) que contiene la clave privada del certificado

### Paso 3 - Verificar el Certificado
Corre el siguiente comando para verificar la salida
```
openssl x509 -in server.crt -text -noout | grep -A1 "Subject Alternative Name"
```
La salida del comando debería ser algo similar
```
            X509v3 Subject Alternative Name:
                DNS:flux-bancogalicia.com, DNS:*.flux-bancogalicia.com
```

### Paso 4 - Instalar el certificado localmente
Como el cerficiado es firmado por uno mismo no tiene entidad cerficadora CA valida o conocida por los user-agents. Para hacer la pruebas locales vas a tener que instalar este certiricado en tu computadora e indicarle si confias el él.

En mac esto se hace desde la aplicación "Acceso a llaveros". Tenes que ir al menú Archivo->Importar elementos y seleccionar el archivo *.crt, en nuestro caso es server.crt. Luego que lo importas, lo buscas, lo abris, desplegas la parte de confiar y selecciona la opcion de confiar siempre.

### Paso 5 - Pasar a Base 64
Si en algún momento necesitas pasar el cerfificado y clave privada a base 64 por ejemplo al usarlo en un secret para kubertentes te dejo de yapa los comandos que copian el contendio de los archivos en base 64 y te quedan listos para hacer Paste donde los necesites (ojo esto son el linux o mac)

```
cat server.key | base64 | pbcopy
cat server.crt | base64 | pbcopy
```

### Paso 6 - Secret
Así es como lo utilizarias en la generación de un secret para de kubertentes

```
apiVersion: v1
kind: Secret
metadata:
  name: banco-galicia-certs
  namespace: banco-galicia
type: kubernetes.io/tls
data:
  tls.key: (pegar el base 64 generado en el paso 5 del key)
  tls.crt: (pegar el base 64 generado en el paso 5 del crt)
```

### Paso 7 - Como utilizarlo TLS en Ingress 
Manifiesto de ingress para un servicio **grpc** que con seguridad tls.
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grpc-ingress
  namespace: banco-galicia
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
    - host: grpc.flux-bancogalicia.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-grpc-service
                port:
                  number: 9090
  tls:
  - hosts:
      - grpc.flux-bancogalicia.com
    secretName: banco-galicia-certs
```

Manifiesto de ingress para un servicio **rest** que con seguridad tls.
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rest-ingress
  namespace: banco-galicia
spec:
  ingressClassName: nginx
  rules:
    - host: rest.flux-bancogalicia.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-grpc-service
                port:
                  number: 8080
  tls:
  - hosts:
      - rest.flux-bancogalicia.com
    secretName: banco-galicia-certs
```

## Paso 8 - DNS Local
Obviamente, como estamos trabajando en un entorno local nuestra pc no va a resolver el dns por eso editamos el archivo /etc/hosts y agregamos la entradas para que resuelvan nuestros dns

```` 
##
127.0.0.1       localhost
255.255.255.255 broadcasthost
::1             localhost
127.0.0.1       rest.flux-bancogalicia.com
127.0.0.1       grpc.flux-bancogalicia.com
````


## Paso 9 - vualá (ojo no confindir con la fintech)
 Porbar el servicio rest con curl

 ```
 curl https://rest.flux-bancogalicia.com/user
 ```
Probar el servico con grpcurl

 ```
 grpcurl  grpc.flux-bancogalicia.com:443 com.flux.grpc.stubs.UserService/getAllUsers
```

Casi seguro que vas a necesitar instal grpcurl

