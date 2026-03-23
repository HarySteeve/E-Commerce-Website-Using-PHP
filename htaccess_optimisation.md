# Optimisations possibles par htaccess

Comme promis, dans la série des optimisations possibles par htaccess, nous parlerons aujourd’hui des optimisations du cache pour tous les types de fichiers, des compatibilités avec IE (oui il y a des astuces à ce niveau-là), la normalisation des types MIME (les identifiants de format de données, je reviendrai dessus), la concaténation des fichiers javascript et css, la compression Gzip, la désactivation des ETag (je reviendrai dessus), etc.

**Conseil :** utilisez les optimisations une par une, ce sont des modifications poussées qui peuvent engendrer comme dans l’article précédent, des erreurs 500.

## Compatibilité Internet Explorer

L’astuce qui suit permet d’améliorer l’affichage de vos pages et l’interprétation des fichiers JavaScript par les versions d’Internet Explorer en forçant le navigateur à utiliser le moteur de rendu le plus rapide et le plus performant disponible. De temps en temps IE utilise un moteur plus ancien pour l’affichage des pages pour par exemple les sites en localhost, les intranets, etc. (avec les dernières versions de IE, il y a un petit icone de page brisée qui s’affiche en haut à droite il me semble).

```apache
<IfModule mod_headers.c>
    Header set X-UA-Compatible "IE=Edge,chrome=1"
    <FilesMatch "\.(js|css|gif|png|jpe?g|pdf|xml|oga|ogg|m4a|ogv|mp4|m4v|webm|svg|svgz|eot|ttf|otf|woff|ico|webp|appcache|manifest|htc|crx|xpi|safariextz|vcf)$">
        Header unset X-UA-Compatible
    </FilesMatch>
</IfModule>
```

## Polices d'écriture

Pour accéder à vos polices d’écriture depuis les sous-domaines de votre site :

```apache
<FilesMatch "\.(ttf|ttc|otf|eot|woff|font\.css)$">
  <IfModule mod_headers.c>
    Header set Access-Control-Allow-Origin "*"
  </IfModule>
</FilesMatch>
```

## Normalisation des types MIME

L’astuce suivant permet de normaliser tous les types MIME des fichiers. Il est possible par une mauvaise configuration du serveur sur lequel vous êtes hébergé que certains fichiers retournent un mauvais type MIME. Qu’est-ce que cela veut dire ? Par exemple, un fichier CSS peut renvoyer un type text/plain ou application/x-pointplus au lieu du bon type. Conséquence : le fichier est tout bonnement ignoré par Mozilla et les navigateurs Netscape.

```apache
# JavaScript
AddType application/javascript         js

# Audio
AddType audio/ogg                      oga ogg
AddType audio/mp4                      m4a

# Video
AddType video/ogg                      ogv
AddType video/mp4                      mp4 m4v
AddType video/webm                     webm

# SVG
AddType image/svg+xml                  svg svgz
AddEncoding gzip                       svgz

# Webfonts
AddType application/vnd.ms-fontobject  eot
AddType application/x-font-ttf         ttf ttc
AddType font/opentype                  otf
AddType application/x-font-woff        woff

# Divers
AddType image/x-icon                   ico
AddType image/webp                     webp
AddType text/cache-manifest            appcache manifest
AddType text/x-component               htc
AddType application/x-chrome-extension crx
AddType application/x-xpinstall        xpi
AddType application/octet-stream       safariextz
AddType text/x-vcard                   vcf
```

## Concaténation des fichiers JS et CSS

Pour mettre en place la concaténation des fichiers JavaScript et css dynamiquement dans des fichiers `script.combined.js` et `script.combined.css`, il faut utiliser le code suivant (pour des questions de gain de performance et de temps de chargement des pages. Je vous conseille de faire quelques recherches sur ce code pour savoir comment l’utiliser) :

```apache
<FilesMatch "\.combined\.js$">
  Options +Includes
  AddOutputFilterByType INCLUDES application/javascript application/json
  SetOutputFilter INCLUDES
</FilesMatch>

<FilesMatch "\.combined\.css$">
  Options +Includes
  AddOutputFilterByType INCLUDES text/css
  SetOutputFilter INCLUDES
</FilesMatch>
```

## Compression GZip

La compression GZip, très utile pour améliorer sensiblement les temps de chargement des pages. En fait, le code suivant va forcer le server à compiler et zipper vos pages pour les afficher plus rapidement. Cela a comme conséquence d’utiliser un peu plus de puissance des processeurs du serveur (ils sont largement inactifs en général) mais en contrepartie vous économisez beaucoup de bande passante, donc j’entends améliorer la vitesse de chargement de pages.

```apache
<IfModule mod_deflate.c>
# Force deflate for mangled headers developer.yahoo.com/blogs/ydn/posts/2010/12/pushing-beyond-gzipping/
<IfModule mod_setenvif.c>
<IfModule mod_headers.c>
SetEnvIfNoCase ^(Accept-EncodXng|X-cept-Encoding|X{15}|~{15}|-{15})$ ^((gzip|deflate)\s*,?\s*)+|[X~-]{4,13}$ HAVE_Accept-Encoding
RequestHeader append Accept-Encoding "gzip,deflate" env=HAVE_Accept-Encoding
</IfModule>
</IfModule>

<IfModule filter_module>
# HTML, TXT, CSS, JavaScript, JSON, XML, HTC:
FilterDeclare COMPRESS
FilterProvider COMPRESS DEFLATE resp=Content-Type $text/html
FilterProvider COMPRESS DEFLATE resp=Content-Type $text/css
FilterProvider COMPRESS DEFLATE resp=Content-Type $text/plain
FilterProvider COMPRESS DEFLATE resp=Content-Type $text/xml
FilterProvider COMPRESS DEFLATE resp=Content-Type $text/x-component
FilterProvider COMPRESS DEFLATE resp=Content-Type $application/javascript
FilterProvider COMPRESS DEFLATE resp=Content-Type $application/json
FilterProvider COMPRESS DEFLATE resp=Content-Type $application/xml
FilterProvider COMPRESS DEFLATE resp=Content-Type $application/xhtml+xml
FilterProvider COMPRESS DEFLATE resp=Content-Type $application/rss+xml
FilterProvider COMPRESS DEFLATE resp=Content-Type $application/atom+xml
FilterProvider COMPRESS DEFLATE resp=Content-Type $application/vnd.ms-fontobject
FilterProvider COMPRESS DEFLATE resp=Content-Type $image/svg+xml
FilterProvider COMPRESS DEFLATE resp=Content-Type $application/x-font-ttf
FilterProvider COMPRESS DEFLATE resp=Content-Type $font/opentype
FilterChain COMPRESS
FilterProtocol COMPRESS DEFLATE change=yes;byteranges=no
</IfModule>

<IfModule !mod_filter.c>
# Legacy versions of Apache
AddOutputFilterByType DEFLATE text/html text/plain text/css application/json
AddOutputFilterByType DEFLATE application/javascript
AddOutputFilterByType DEFLATE text/xml application/xml text/x-component
AddOutputFilterByType DEFLATE application/xhtml+xml application/rss+xml application/atom+xml
AddOutputFilterByType DEFLATE image/svg+xml application/vnd.ms-fontobject application/x-font-ttf font/opentype
</IfModule>
</IfModule>
```

## Mise en cache

Chaque fichier sur une page web renvoie une entête au client, à votre navigateur. L’entête contient notamment la date d’expiration des fichiers. Spécifier une date d’expiration d’un fichier a pour conséquence la mise en cache de celui-ci. Par défaut, elle n’est pas spécifiée, donc les fichiers ne sont pas mis en cache. Voici comment procéder pour modifier les entêtes des fichiers et donc modifier la mise en cache de tous types de médias :

```apache
<IfModule mod_expires.c>
ExpiresActive on

# Perhaps better to whitelist expires rules? Perhaps.
ExpiresDefault "access plus 1 month"

# cache.appcache needs re-requests in FF 3.6 (thanks Remy ~Introducing HTML5)
ExpiresByType text/cache-manifest "access plus 0 seconds"

# Your document html
ExpiresByType text/html "access plus 0 seconds"

# Data
ExpiresByType text/xml "access plus 0 seconds"
ExpiresByType application/xml "access plus 0 seconds"
ExpiresByType application/json "access plus 0 seconds"

# Feed
ExpiresByType application/rss+xml "access plus 1 hour"
ExpiresByType application/atom+xml "access plus 1 hour"

# Favicon (cannot be renamed)
ExpiresByType image/x-icon "access plus 1 week"

# Media: images, video, audio
ExpiresByType image/gif "access plus 1 month"
ExpiresByType image/png "access plus 1 month"
ExpiresByType image/jpg "access plus 1 month"
ExpiresByType image/jpeg "access plus 1 month"
ExpiresByType video/ogg "access plus 1 month"
ExpiresByType audio/ogg "access plus 1 month"
ExpiresByType video/mp4 "access plus 1 month"
ExpiresByType video/webm "access plus 1 month"

# HTC files (css3pie)
ExpiresByType text/x-component "access plus 1 month"

# Webfonts
ExpiresByType font/truetype "access plus 1 month"
ExpiresByType font/opentype "access plus 1 month"
ExpiresByType application/x-font-woff "access plus 1 month"
ExpiresByType image/svg+xml "access plus 1 month"
ExpiresByType application/vnd.ms-fontobject "access plus 1 month"

# CSS and JavaScript
ExpiresByType text/css "access plus 1 year"
ExpiresByType application/javascript "access plus 1 year"

<IfModule mod_headers.c>
Header append Cache-Control "public"
</IfModule>
</IfModule>
```

## Désactivation des ETags

La désactivation de l’ETag, c’est un peu complexe mais pour savoir pourquoi il faut le désactiver, une recherche rapide sur google vous renverra a la bonne adresse. Pour faire bref, un ETag sert au serveur web à identifier une ressource et sa version.

```apache
<IfModule mod_headers.c>
  Header unset ETag
</IfModule>
```

## Autoriser les cookies depuis une iFrame

Une autre directive htaccess qui peut servir. Elle permet d’autoriser la mise en place de cookie à partie d’une iframe. Il faut cependant compléter le code avec votre domaine et y uploader le fichier `p3p.xml`.

```apache
<IfModule mod_headers.c>
<Location />
Header set P3P 'policyref="http://www.votre-domaine.com/w3c/p3p.xml", CP="IDC DSP COR ADM DEVi TAIi PSA PSD IVAi IVDi CONi HIS OUR IND CNT"'
</Location>
</IfModule>
```

## Forcer le HTTPS

Voici un code pour éviter les warnings que votre site peu afficher, toujours ennuyant pour vos clients :

```apache
<IfModule mod_rewrite.c>
RewriteCond %{SERVER_PORT} !^443
RewriteRule ^ https://www.votre-domaine.com%{REQUEST_URI} [R=301,L]
</IfModule>
```

## Encoder les fichiers en UTF-8

Et une petite dernière pour encoder tous vos fichiers en UTF8 :

```apache
AddDefaultCharset utf-8
AddCharset utf-8 .html .css .js .xml .json .rss .atom
```

---

## Directives de redirection SEO

Pour des modifications un peu plus SEO, voici une liste de directives que j’utilise assez souvent.

### Redirection de l'index.php vers la racine (`/`)

```apache
RewriteCond %{THE_REQUEST} ^[A-Z]{3,9}\ /.*index\.php\ HTTP/
RewriteRule ^(.*)index\.php$ /$1 [R=301,L]
```
ou
```apache
RewriteRule ^index\.php$ http://www.votresite.com/ [QSA,L,R=301]
```

### Redirection du domaine sans `www` vers les `www`

```apache
RewriteCond %{HTTP_HOST} ^exemple.com$
RewriteRule ^(.*) http://www.exemple.com/$1 [QSA,L,R=301]
```
ou
```apache
RewriteCond %{HTTP_HOST} !^www\.site\.fr$
RewriteRule ^(.*) http://www.site.fr/$1 [QSA,L,R=301]
```

### Redirection de `domaine1` vers `domaine2`

```apache
RewriteCond %{HTTP_HOST} ^domaine1\.com$
RewriteRule ^(.*) http://www.domaine2.com/$1 [QSA,L,R=301]
```

### Gestion des erreurs 404

```apache
ErrorDocument 404 /404.php
```