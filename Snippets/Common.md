#### Exiftool

```shell
# Convert photo files
LC_ALL="ru_RU.UTF-8" exiftool '-Directory<CreateDate' '-Directory<ProfileDateTime' '-Directory<DateTimeOriginal' '-Directory<FileModifyDate' '-filename=%f.%le' -ext jpg -d ~/Pictures/Photo/%Y/%B -r IphonePhoto/
# Convert video files
LC_ALL="ru_RU.UTF-8" exiftool -ee '-Directory<CreationDate' '-Directory<CreateDate' '-filename=%f.%le' -d Video/%Y/%B -r video/
```

#### Mogrify

```bash
# Reduce size for all jpg in the current directory
mogrify -path . -resize 1920x1920 -quality 70 *.jpg
# Resize and convert to png
mogrify -path . -resize 640x480 -format png -quality 70 *.jpg
mogrify -path . -resize 320x240 -format png -quality 70 *.png
# Move image to East and set resolution
mogrify -extent 640x480 -gravity East -background none *

magick mogrify -resize "314x332" -gravity center -extent 314x332 -background transparent -flatten *
magick mogrify -background transparent -gravity west -extent 634x454 -gravity west -splice 6x0 -gravity south -splice 0x26 *
```

#### Ffmpeg

Convert all mkv files and reduce size by 2
```shell
#!/bin/bash
for i in *.MOV; do
  ffmpeg -i "$i" -map_metadata 0 -vcodec h264 -acodec mp2 "${i%.*}.mp4";
  # rm -rf "$i"
  # ffmpeg -i "$i" -vf "scale=trunc(iw/4)*2:trunc(ih/4)*2" -c:v libx265 -crf 28 "${i%.*}.mp4";
done
```

#### Nginx cache

```nginx
upstream toptop {
    server localhost:4001 fail_timeout=0 weight=5;
}
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=one:10m inactive=24h  max_size=1g;
    proxy_temp_path /var/cache/nginx/tmp;

server {
    listen       80;
    server_name  toptop.ru;

    proxy_cache one;

    access_log /var/log/nginx/toptop.access.log combined;
    error_log /var/log/nginx/toptop.error.log info;

    root /var/www/toptop/current/public;

    location / {
      try_files $uri @app;
    }

    location ~* \.(?:css|js)$ {
      expires 1y;
      access_log off;
      add_header Cache-Control "public";
    }

    location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc)$ {
      expires 1M;
      access_log off;
      add_header Cache-Control "public";
    }

    location @app {
      proxy_pass http://toptop;
    }
}

```

#### Bookmarklet
https://caiorss.github.io/bookmarklet-maker/
```js
javascript: Promise.all([import('https://unpkg.com/turndown@7.1.2?module'), import('https://unpkg.com/@tehshrike/readability@0.2.0'), ]).then(async ([{
    default: Turndown
}, {
    default: Readability
}]) => {

  /* Optional vault name */
  const vault = "";

  /* Optional folder name such as "Clippings/" */
  const folder = "Raw/Notes/";

  /* Optional tags  */
  let tags = "";

  /* Parse the site's meta keywords content into tags, if present */
  if (document.querySelector('meta[name="keywords" i]')) {
      var keywords = document.querySelector('meta[name="keywords" i]').getAttribute('content').split(',');

      keywords.forEach(function(keyword) {
          let tag = ' ' + keyword.split(' ').join('');
          tags += tag;
      });
  }

  function getSelectionHtml() {
    var html = "";
    if (typeof window.getSelection != "undefined") {
        var sel = window.getSelection();
        if (sel.rangeCount) {
            var container = document.createElement("div");
            for (var i = 0, len = sel.rangeCount; i < len; ++i) {
                container.appendChild(sel.getRangeAt(i).cloneContents());
            }
            html = container.innerHTML;
        }
    } else if (typeof document.selection != "undefined") {
        if (document.selection.type == "Text") {
            html = document.selection.createRange().htmlText;
        }
    }
    return html;
  }

  const selection = getSelectionHtml();

  const {
      title,
      byline,
      content
  } = new Readability(document.cloneNode(true)).parse();

  function getFileName(fileName) {
    var userAgent = window.navigator.userAgent,
        platform = window.navigator.platform,
        windowsPlatforms = ['Win32', 'Win64', 'Windows', 'WinCE'];

    if (windowsPlatforms.indexOf(platform) !== -1) {
      fileName = fileName.replace(':', '').replace(/[/\\?%*|"<>]/g, '-');
    } else {
      fileName = fileName.replace(':', '').replace('|', '').replace(/\//g, '-').replace(/\\/g, '-');
    }
    return fileName;
  }

  const fileName = getFileName(title);

  if (selection) {
      var markdownify = selection;
  } else {
      var markdownify = content;
  }

  if (vault) {
      var vaultName = '&vault=' + encodeURIComponent(`${vault}`);
  } else {
      var vaultName = '';
  }

  const markdownBody = new Turndown({
      headingStyle: 'atx',
      hr: '---',
      bulletListMarker: '-',
      codeBlockStyle: 'fenced',
      emDelimiter: '*',
  }).turndown(markdownify);

  var date = new Date();

  function convertDate(date) {
    var yyyy = date.getFullYear().toString();
    var mm = (date.getMonth()+1).toString();
    var dd  = date.getDate().toString();
    var mmChars = mm.split('');
    var ddChars = dd.split('');
    return yyyy + '-' + (mmChars[1]?mm:"0"+mmChars[0]) + '-' + (ddChars[1]?dd:"0"+ddChars[0]);
  }

  const today = convertDate(date);

  // Utility function to get meta content by name or property
  function getMetaContent(attr, value) {
      var element = document.querySelector(`meta[${attr}='${value}']`);
      return element ? element.getAttribute("content").trim() : "";
  }

  // Fetch byline, meta author, property author, or site name
  var author = byline || getMetaContent("name", "author") || getMetaContent("property", "author") || getMetaContent("property", "og:site_name");

  /* Try to get published date */
  var timeElement = document.querySelector("time");
  var publishedDate = timeElement ? timeElement.getAttribute("datetime") : "";

  if (publishedDate && publishedDate.trim() !== "") {
      var date = new Date(publishedDate);
      var year = date.getFullYear();
      var month = date.getMonth() + 1; // Months are 0-based in JavaScript
      var day = date.getDate();

      // Pad month and day with leading zeros if necessary
      month = month < 10 ? '0' + month : month;
      day = day < 10 ? '0' + day : day;

      var published = year + '-' + month + '-' + day;
  } else {
      var published = ''
  }

  /* YAML front matter as tags render cleaner with special chars  */
  const fileContent = 
      '---\n'
      + 'title: "' + title + '"\n'
      + 'source: ' + document.URL + '\n'
      + 'clipped: ' + today + '\n'
      + 'published: ' + published + '\n' 
      + 'category: \n'
      + 'tags: [' + tags + ']\n'
      + 'read: false\n' 
      + '---\n\n'
      + markdownBody ;

   document.location.href = "obsidian://new?"
    + "file=" + encodeURIComponent(folder + fileName)
    + "&content=" + encodeURIComponent(fileContent)
    + vaultName ;

})
```
#### SDCard backup & restore
```shell
sudo dd bs=4M conv=sync,noerror if=/dev/rdisk4 status=progress | gzip > kodi.img.gz # Backup 
sudo gzip -dc ./kodi.img.gz | sudo dd conv=sync,noerror bs=4M of=/dev/rdisk4 status=progress # Restore
```
#### LibreELEC
```shell
gunzip -d LibreELEC-RPi4.arm-11.0.3.img.gz
sudo dd conv=fsync bs=4M if=./LibreELEC-RPi4.arm-11.0.3.img of=/dev/rdisk6 status=progress
```

#### MySQL Snippet
```bash
# login to mysql 
mysql -u platformsupportchat -h proxysql.platform.mysql.tutu.ru --port 6401 -p
# SHOW TABLES;
```

#### Golang

```shell
# Clean modcache
go clean -modcache
```

#### Zig

https://github.com/BrookJeynes/zig-fetch/tree/main