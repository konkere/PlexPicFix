# Plex > Решение проблемы отображения постеров и других картинок

Проблема состоит из двух частей.

## 1. Проблема (часть) первая.

[DNS](https://ru.wikipedia.org/wiki/DNS) резолвят [127.0.0.1](https://ru.wikipedia.org/wiki/Localhost) на домены с картинками (ещё и не постоянно) вместо реального IP-адреса.

![](/screenshots/dig_ito.jpg)

Эта часть проблемы решается двумя способами:

**1.1.** Установкой DNS, которые резолвят только правильные IP. Например DNS от [Quad9](https://www.quad9.net/): `9.9.9.9` и `149.112.112.112`;

**1.2.** Заворачиванием в [VPN](https://ru.wikipedia.org/wiki/VPN) трафика на адреса используемых DNS (если они не провайдерские).



## 2. Проблема (часть) вторая.

Запрет доступа к [CDN](https://ru.wikipedia.org/wiki/Content_Delivery_Network) с картинками по [403 ошибке](https://ru.wikipedia.org/wiki/%D0%9E%D1%88%D0%B8%D0%B1%D0%BA%D0%B0_403). Plex использует две CDN: Amazon CloudFrontCDN и BunnyCDN. И если ко второй доступ свободный и беспрепятственный, то CloudFront с большой вероятностью блокирует к себе доступ (403 Forbidden). 

![](/screenshots/cf403.jpg)

Здесь всё может оказаться сложнее. Есть несколько рабочих способов. Перечислю по мере усложнения и маргинальности, а также отмечу, что начальные могут не у всех сработать. Мне, например, помог только последний.

**2.1.** Завернуть в VPN маршруты до доменов `image.tmdb.org`, `api.themoviedb.org`, `themoviedb.org`, `www.themoviedb.org` (обращу внимание на то, что они все на Amazon и их IP постоянно меняются, так что возможны сбои).

![](/screenshots/routes_ito.png)

**2.2.** Завернуть в VPN одну из подсетей CloudFront:
```
NetRange:  108.138.0.0 - 108.139.255.255
CIDR:   108.138.0.0/15
NetName:  AMAZON-CF
NetHandle:  NET-108-138-0-0-1
```

**2.3.** Если есть свой DNS (на роутере почти наверняка), прописать [CNAME](https://ru.wikipedia.org/wiki/DNS#%D0%97%D0%B0%D0%BF%D0%B8%D1%81%D0%B8_DNS) `image.tmdb.org` для `tmdb-image-prod.b-cdn.net`

![](/screenshots/cname_ito.png)

**2.4.** Завернуть в VPN трафик на **ВСЕ** адреса CloudFront:

**2.4.1.** путём создания на роутере списков ip-адресов, маркировки трафика на эти адреса и направления маршрута промаркированного трафика в туннель VPN.

![](/screenshots/cf_mark.png)

Адреса не статичные, могут и будут со временем меняться. Актуальный список адресов находится [на сайте CloudFront](https://d7uri8nf7uskq.cloudfront.net/tools/list-cloudfront-ips). Для автоматического обновления списков я написал себе [скриптик](https://github.com/konkere/pyRouterOSviaSSH/blob/main/mikrotik_addrlist_upd.py) (работает на [Python](https://ru.wikipedia.org/wiki/Python) в связке с Mikrotik [RouterOS](https://ru.wikipedia.org/wiki/MikroTik#RouterOS), гвоздями ничего не прибивал, можно использовать и для других списков, не только CloudFront).

**2.4.2.** путём организации маршрутизации средствами [BGP](https://ru.wikipedia.org/wiki/Border_Gateway_Protocol) (если роутер поддерживает работу с этим протоколом) и направлении трафика на эти адреса через туннель VPN.

![](/screenshots/af_bgp.png)

Например: [Antifilter](https://antifilter.network/bgp). Маршруты и списки будут обновляться автоматически.

**2.4.3.** добрые люди поделились ссылкой на статью «[AdGuard Home для выборочного обхода блокировок](https://forum.keenetic.com/topic/16441-adguard-home-%D0%B4%D0%BB%D1%8F-%D0%B2%D1%8B%D0%B1%D0%BE%D1%80%D0%BE%D1%87%D0%BD%D0%BE%D0%B3%D0%BE-%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF%D0%B0-%D0%BA-%D0%B7%D0%B0%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D0%BC-%D0%B4%D0%BE%D0%BC%D0%B5%D0%BD%D0%B0%D0%BC/)» (+[для IPv6](https://forum.keenetic.com/topic/16441-adguard-home-%D0%B4%D0%BB%D1%8F-%D0%B2%D1%8B%D0%B1%D0%BE%D1%80%D0%BE%D1%87%D0%BD%D0%BE%D0%B3%D0%BE-%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF%D0%B0-%D0%BA-%D0%B7%D0%B0%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D0%BC-%D0%B4%D0%BE%D0%BC%D0%B5%D0%BD%D0%B0%D0%BC/page/2/#comment-169381)), которую можно спроецировать на наш частный случай. Автор статьи все эти шаманства делает для роутера Keenetic, но, судя по использованию [opkg](https://help.keenetic.com/hc/ru/articles/360000948719-OPKG), должно работать на любом маршрутизаторе, на котором возможен запуск данного пакетного менеджера (например, если прошить роутер [OpenWRT](https://ru.wikipedia.org/wiki/OpenWrt) или [DD-WRT](https://ru.wikipedia.org/wiki/DD-WRT)).



## Дополнительная информация.

О том, как вернуть уже побитые картинки, ведь даже после вышеописанных действий они сами этого не сделают, ибо локально закэшировались в таком виде и Plex уверен, что всё хорошо.

Нужно либо временно перенести фильмы/сериалы из каталогов (сами файлы), сканируемых Plex'ом, нажать в библиотеках «Очистить корзину»

![](/screenshots/plex_cleanrec.png)

и после этого вернуть файлы обратно. Или нажать на каждом «Не соответствует» с последующей очисткой корзины.

![](/screenshots/plex_not.png)

После этого можно заново сопоставить все эти фильмы и сериалы, картинки подтянутся свежи и пяни.

**ЗЫ** вместе с очисткой корзины я ещё на всякий случай жму в общих настройках Plex две эти кнопки (в обратном порядке):

![](/screenshots/plex_cleanbd.png)