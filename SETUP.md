# Setup

Dois comandos. Precisas de **Docker** e **git** — mais nada. Não é preciso PHP nem
composer instalados; o PHPUnit já vem no repositório.

## 1. Arrancar a aplicação

```bash
docker compose -f docker-compose.sqlite.yml up -d
```

Abre **http://127.0.0.1** — utilizador `admin`, senha `admin`.

> **Usa `127.0.0.1`, não `localhost`.** Se já correste outros projetos em `localhost`,
> o browser tem cookies acumulados desse domínio e o nginx do contentor responde
> `400 Bad Request — Request Header Or Cookie Too Large`. O `127.0.0.1` é outro host
> para efeitos de cookies, por isso vai limpo. (Em alternativa: janela privada.)

> Se a porta 80 estiver ocupada, muda `"80:80"` para `"8080:80"` no
> `docker-compose.sqlite.yml` e usa http://127.0.0.1:8080

Para parar: `docker compose -f docker-compose.sqlite.yml down`

## 2. Correr os testes

```bash
docker run --rm -v "$PWD":/srv/work/kanboard -w /srv/work/kanboard php:8.4-cli \
  php vendor/bin/phpunit -c tests/units.sqlite.xml tests/units/Model
```

Esperado: **OK (465 tests, 5053 assertions)**, cerca de 2m30.

Um ficheiro só, que é o ciclo normal de trabalho (~4 segundos):

```bash
docker run --rm -v "$PWD":/srv/work/kanboard -w /srv/work/kanboard php:8.4-cli \
  php vendor/bin/phpunit -c tests/units.sqlite.xml --filter ColumnModelTest
```

---

## Duas armadilhas

**Monta sempre em `/srv/work/kanboard`, nunca em `/app`.**
Há um teste que faz `dirname(getcwd())`. Se o repositório estiver montado noutro sítio,
o `FunctionTest::testSanitizePath` falha sem razão aparente.

**Não corras a suite inteira.**
Ao teste 546 rebenta num teste de LDAP — falta a extensão `ldap` no PHP de prateleira —
e o `tests/units.sqlite.xml` tem `stopOnFailure="true"`, por isso aborta tudo. Não
partiste nada. Usa `tests/units/Model`, que é onde trabalhamos.

---

## Verificar contra uma versão de PHP

O Docker dá a versão de destino de graça: corre-se o mesmo código dentro da imagem
dessa versão.

```bash
# o código passa no PHP 8.1 (o mínimo que o projeto declara)
docker run --rm -v "$PWD":/w -w /w php:8.1-cli \
  sh -c 'find app -name "*.php" -print0 | xargs -0 -n40 -P4 php -l' | grep -v "No syntax errors"

# e não passa no 7.4
docker run --rm -v "$PWD":/w -w /w php:7.4-cli \
  sh -c 'find app -name "*.php" -print0 | xargs -0 -n40 -P4 php -l' | grep "Parse error"
```

O segundo devolve:

```
Parse error: syntax error, unexpected '|', expecting '{' in app/functions.php on line 307
```

É um *union type* (`int|string`), sintaxe que só existe a partir do PHP 8.0.
