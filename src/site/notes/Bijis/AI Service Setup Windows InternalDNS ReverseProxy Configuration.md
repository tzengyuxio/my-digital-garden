---
{"dg-publish":true,"dg-path":"AI Service Setup Windows InternalDNS ReverseProxy Configuration.md","permalink":"/AI Service Setup Windows InternalDNS ReverseProxy Configuration/","title":"區網 AI 服務管理筆記：DNS 與反向代理實踐","tags":["dify","ollama","comfyui","reverse-proxy","open-webui","caddy","winsw","dns"],"noteIcon":"","created":"2024-10-03T19:49:30.481+08:00","updated":"2024-10-14T13:19:05.330+08:00"}
---


## Preface

在 Windows PC 上自架了 [ComfyUI](https://github.com/comfyanonymous/ComfyUI), [Dify](https://dify.ai/) 與 [Open WebUI](https://openwebui.com/) 等 AI 服務來玩玩，但我習慣還是在 Mac 上作業，所以平常都是從 Mac 開瀏覽器打開 `192.168.12.34:3000` 這類的網址連到 PC 上，不同的服務就走不同的 port。

但一來這網址不好記，其次不夠專業，再說我也想把上面這些服務以簡單的方式給區網內的家人都可以使用，或是用手機平板也能使用。因此有了以下想法目標：

1. 用 domain name 取代 IP
2. 不同的服務走不同的 domain name, 例如:
	1. `192.168.12.34:8188` -> `comfyui.home.local`
	2. `192.168.12.34:8080` -> `dify.home.local`

這個其實是平常我們上網時網路機制運作的日常，但要在區網內搞定，其實也牽涉了不少知識點與技術選擇。這邊留個筆記，方便自己日後查看。

## 基本概念

- 在內網架 DNS 來做名稱解析
- 在 PC 上跑反向代理 (reverse proxy) 來將不同的網址轉到不同的服務上。

## 過程紀錄

### DNS 安裝與相關設定

1. 首先是 DNS。家裡網路有台 Synology NAS，看了一下有 DNS 的功能，進管理介面安裝一下套件即可。
2. 裝完設定 DNS。手邊有閒置的域名，不過這些服務我只打算在內網使用，因此就簡單取了個 `home.local` 作為域名，並且建好幾筆資料全部指向 `192.168.12.34` (仮) 這個 IP。
3. 再來就是要改其他設備的 DNS 主機了
	1. ~~最初我是改路由器上的 DNS 設定，主要 DNS 設為 Synology NAS, 次要設為寬頻服務商的 DNS~~
		1. 這個方法能用名字連到內網機器，但外網都連不到。因為主要 DNS 服務正常，所以不會再去問次要 DNS，但主要 DNS 只有內網資料。
	2. 正確的做法應該是改**路由器的 DHCP 設定**，將設定中的主要 DNS 設定為 Synology NAS 這台，次要留空。
		1. 另外要在 NAS DNS 設定中啟用「解析服務」，以及「啟動轉寄站」，轉寄站設定為上游寬頻服務商的 DNS。
		2. 設定在 DHCP 的好處是，當區網內有其他設備連上網時，便會從 DHCP Server 拿到 DNS 設定，就不用一台一台設備去改了。

到這邊為止，使用網址就可以連到區網內的 PC 了。

### 反向代理

簡單查了一下 Windows 上的反向代理 solution, 有以下幾個：
- `Nginx`
- `IIS`
- `Caddy`

我之前在另一台外網的 Debian server 上有用過 [Traefik](https://traefik.io/) 來做反向代理，使用上蠻滿意的。但我不想在 Windows 上額外裝太多東西，所以最初的想法是 IIS，因為系統自帶，只要開啟功能就好。但研究了一下發現 IIS 的反向代理需要下載安裝模組，剛好又看到 [Caddy](https://caddyserver.com/) 只有單一一個執行檔，非常綠色且輕量，於是這次就用它了。

Caddy 的設定可以直接參考官方文件，這邊不多說。倒是在設定反向代理時遇到些問題，問題本身不複雜也不困難，就是查找費勁。

1. 首先是 `80/443` port 被佔走了，所以 Caddy 跑不起來。最初找到是 docker 佔走，以為是什麼服務。後來才發現是 `dify` container 預設就是用了 80 port。所以還得先改 `dify` 的 `docker-compose` file 將其 port 改至 `8080/8443`。
2. 還有 HTTPS 證書問題。設定完後打開網頁，Chrome 會跳出 `net::ERR_CERT_AUTHORITY_INVALID` （你的連線不是私人連線）的錯誤訊息。因為用的不是正式註冊的域名，所以 Caddy 會生成一個私有的「根憑證」，檔案位置在 `%AppData%\Roaming\Caddy\pki\authorities\local`，要將 `root.crt` 拷貝出來加入到客戶端電腦鑰匙圈中，並設定為**永遠信任**。

### 後續小工作

到這邊差不多就告一段落了，不過後面使用上還有一些小插曲，也附記於此。

#### Caddy 開機自動啟動

要能開機自動啟動，只要把 Caddy 註冊為系統服務即可。同樣有幾個方案，後來選了 [`WinSW`](https://github.com/winsw/winsw)，這也是單一一個執行檔加設定檔便可搞定。

#### Dify 的 Ollama 設定

`Dify` 本身可以連很多 LLM 服務，包括本地端或自架的的 `Ollama`，只要設定服務網址即可。

由於 `Dify` 是跑在 Docker 環境裡的，而 `Ollama` 則是 Windows 背景服務程式，兩者的網路環境是不通的。所以一開始不管我設 `192.168.12.34:11434` 或 `127.0.0.1:11434` 都行不通。後來查到 Docker 有個特殊的 domain 可以用來指宿主 (host) 的機器，只要填入 `http://host.docker.internal:11434` 即可。

## 總結

這篇筆記的內容很雜，感覺很多部分可以單獨生一篇出來展開，但這樣寫起來又累又瑣碎。畢竟是寫給自己看為主的，所以很多部分可能會省略掉，包括遇到問題的排查細節，如何使用 CLI tool 來判斷各種網路使用狀況等。

颱風天的突然假期第一天主要耗在了這上面，雖然我沒有颱風假，但是兩天的颱風假還是挺難得一見的。以此為記。