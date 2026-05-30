# plantilla nginx + badbotblocker

el proposito de esta plantilla es facilitar el despliegue de servicios usando dockers de nginx documentado en mi wiki

## quehaceres

- TODO: documentar exhaustivamente
- TODO: encontrar imagen de nginx adecuada -qbit usa alpine-
- TODO: probar en produccion
- TODO: crear un script que descargue el `globalblacklist.conf` de 3b 

```
wget https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/conf.d/globalblacklist.conf -O /etc/nginx/conf.d/globalblacklist.conf
```

- IMPORTANT: como chucha coordino todo esto?

# como usar? (bosquejo)

este proyecto bebe directamente del titanico trabajo de [mitchell krog](mitchellkrog@gmail.com) 

## estructura del repo

- `bots.d`: contiene las configuraciones de 3b
- `conf.d`: cargado al final de `http` de `nginx.conf`, 
- `locate.d`: contiene las reglas de paths genericos a rechazar indistintamente
- `http.d`: proveniente de las empaquetaciones de nginx de alpine, contiene un subdirectorio por cada server con su respectivo `.conf` y `proxy.conf` de ser necesario.
    - [services.d](https://github.com/concienciadesventurada/services.d) contiene esta misma estructura
- `modules`: directorio generico para los modulos de nginx, actualmente sin uso

## enfoque del hardening

en gran parte, bloquea practicamente muchas cosas que [nginx bad bot blocker](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker).


para cualquier modificacion de esta plantilla, los contenidos del directorio `bots.d` son el punto de entrada, pero en la practica, todas ellas son leidas y aplicadas antes que las `conf.d/globalblacklist.conf`:

- `blockbots.conf`: de los `map` que contienen todo el resto de .confs en el `http` scope, este archivo contiene la logica que nginx va a aplicar;
- `blacklist-ips.conf`: listas de ips personalizadas, tambien puede sobreescribir declaraciones de 
- `ddos.conf`: zonas y limites para nginx, frecuentemente declaradas directamente en `http`, la generica declarada en `nginx.conf` debiese ser suficiente por las restricciones que ya tiene el trafico
