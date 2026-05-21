# Тестирование

## Стенд

3 nginx (reverse proxy) + traefik/whoami. Сеть 172.20.0.0/24, адреса фиксированы.

- 8081 → nginx1 → app
- 8082 → nginx2 → nginx1 → app
- 8083 → nginx3 → nginx2 → nginx1 → app

## Решение

`geo` — помечает IP nginx1–3 как доверенные (172.20.0.1 туда не входит).  
`map` — если источник доверенный, дописывает `$remote_addr` к XFF; иначе перезаписывает.

Конфиг один на всех, апстрим берётся из `UPSTREAM_TARGET` через envsubst.

## Запуск

```
docker compose up -d
```

## Тесты

### 1. Один прокси

```
$ curl -s http://localhost:8081
X-Forwarded-For: 172.20.0.1
RemoteAddr: 172.20.0.11:59950
```

### 2. Два прокси

```
$ curl -s http://localhost:8082
X-Forwarded-For: 172.20.0.1, 172.20.0.12
RemoteAddr: 172.20.0.11:59962
```

### 3. Три прокси

```
$ curl -s http://localhost:8083
X-Forwarded-For: 172.20.0.1, 172.20.0.13, 172.20.0.12
RemoteAddr: 172.20.0.11:59978
```

### 4. Спуфинг, один прокси

```
$ curl -s -H "X-Forwarded-For: 1.1.1.1" http://localhost:8081
X-Forwarded-For: 172.20.0.1
```

### 5. Спуфинг, три прокси

```
$ curl -s -H "X-Forwarded-For: 1.1.1.1" http://localhost:8083
X-Forwarded-For: 172.20.0.1, 172.20.0.13, 172.20.0.12
```

## Логи

```
$ docker compose logs nginx1 | tail -5
172.20.0.1  xff_in="-"                          xff_out="172.20.0.1"
172.20.0.12 xff_in="172.20.0.1"                 xff_out="172.20.0.1, 172.20.0.12"
172.20.0.12 xff_in="172.20.0.1, 172.20.0.13"    xff_out="172.20.0.1, 172.20.0.13, 172.20.0.12"
172.20.0.1  xff_in="1.1.1.1"                    xff_out="172.20.0.1"
172.20.0.12 xff_in="172.20.0.1, 172.20.0.13"    xff_out="172.20.0.1, 172.20.0.13, 172.20.0.12"
```
