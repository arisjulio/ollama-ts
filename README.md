# Ollama + Tailscale

Mi archivo "compose" para montar
[Ollama](https://github.com/ollama/ollama/blob/main/docs/docker.md) y
[Tailscale](https://tailscale.com/kb/1282/docker) en contenedores con Docker.

Se aprovecha Tailscale para obtener un certificado SSL y un subdominio. Los
contenedores se montan y funcionan como un nodo de Tailscale, así que aparecerá
una máquina con el nombre "ollama" en la lista de nodos de Tailscale. Solo podrá
ser accedido a través de la [Tailnet](https://tailscale.com/kb/1136/tailnet), no
se expone ningún puerto en el "host".

El contenedor de Ollama utiliza la imagen
[`rocm`](https://github.com/ollama/ollama/blob/main/docs/docker.md#amd-gpu),
que corresponde a la variante que aprovecha las GPU de AMD. El uso de la GPU por
parte de un contenedor de Docker es sencillo cuando el sistema "host" es
GNU/Linux y la GPU es AMD. Dudo que este archivo "compose" funcione en otros
sistemas "host", y para trabajar con GPU NVIDIA hay que revisar la documentación
de Ollama.

## Correr los contenedores

1. Genera una [llave de un solo uso](https://tailscale.com/kb/1085/auth-keys)
para autenticar el nodo en Tailscale.
2. Corre los contenedores con Docker Compose: 
  `TS_AUTHKEY=<your_one_time_auth_key> docker compose up -d`. Reemplaza 
  `<your_one_time_auth_key>` por la llave de un solo uso que obtuviste en el
  paso anterior.
3. Revisa la
  [lista de máquinas de Tailscale](https://login.tailscale.com/admin/machines).
  Debe aparecer una máquina con el nombre "ollama".

## Utilizar Ollama

1. Corre el primer modelo desde la línea de comandos.
  [Gemma3](https://ollama.com/library/gemma3) es un buen candidato para probar
  en chat: `docker exec -it ollama ollama run gemma3`. Esto descargará (si no
  está descargado) el modelo y lo correrá para poder interactuar con él.
2. Busca más modelos en la lista de
   [modelos disponibles](https://ollama.com/search). Instala los que desees, por
   ejemplo llama3.1: `docker exec -it ollama ollama pull llama3.1`.

### Ollama API

Ollama ofrece una [API](https://github.com/ollama/ollama/blob/main/docs/api.md)
(al estilo OpenAI), que permite interactuar con los modelos instalados. Para
usarla, puedes hacer peticiones HTTP a la dirección de tu máquina Ollama en tu
Tailnet. Por ejemplo:

```sh
curl https://ollama.<your-tailnet-domain>/api/generate -d '{
  "model": "gemma3",
  "prompt": "Hola, ¿cómo estás?",
  "stream": false
}' | jq .
```

Reemplaza `<your-tailnet-domain>` con el nombre de tu Tailnet. Se encuentra en
la sección [DNS](https://login.tailscale.com/admin/dns) de la consola de
Tailscale. Suele tener la forma `taila1b2c3.ts.net`.

La petición se hace usando el protocolo `https://`. Si es la primera vez que se
hace una petición al dominio de la máquina "ollama", puede tardar unos segundos
mientras se genera el certificado SSL. Para que esto funcione, MagicDNS y
los certificados HTTPS deben estar habilitados en tu Tailnet, en la sección de
[DNS](https://login.tailscale.com/admin/dns).

## Copias de seguridad

Los modelos de Ollama se almacenan en un volumen llamado `ollama_ollama`, como
se define en el archivo "compose". Estos modelos siempre pueden descargarse
usando el comando `ollama pull`. Sin embargo, si deseas guardar una copia de los
modelos para reutilizarlos en otra instancia de Ollama, o para restaurarlos en
caso de re-despliegue, puede hacerse con un contenedor auxiliar, así:

```sh
docker run --rm --volumes-from ollama -v $(pwd):/backup busybox tar cvf /backup/backup.tar /root/.ollama
```

Este comando crea un archivo tar con todo el contenido del volumen y lo guarda
en el directorio actual con el nombre `backup.tar`.

## Remover Ollama

1. Remover los contenedores: `docker compose down`
2. Remover los volúmenes: `docker volume rm ollama_ollama ollama_ts-authkey`
3. [Remover la máquina](https://tailscale.com/kb/1260/device-remove#remove-devices-from-the-admin-console)
"ollama" de la Tailnet.

### Restaurar Ollama

Si hiciste una una copia del volumen `ollama_ollama` para poder restaurarlo
después de haber borrado los contenedores y volúmenes, puedes hacerlo siguiendo
los pasos:

1. Vuelve a [correr los contenedores](#correr-los-contenedores). Asegúrate de
  que la máquina "ollama" aparezca de nuevo en tu Tailnet.
2. Restaura el volumen `ollama_ollama` con el archivo tar que guardaste antes.
  Esto puede hacerse con un contenedor auxiliar, así:
    ```sh
    docker run --rm --volumes-from ollama -v $(pwd):/backup busybox tar xvf /backup/backup.tar
    ```
3. Corre un modelo que sepas que debería estar descargado. Por ejemplo, Gemma3:
  `docker exec -it ollama ollama run gemma3`. El modelo no debería descargarse,
  pues fue restaurado de la copia de seguridad, así que debería ejecutarse
  enseguida.
