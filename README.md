# web-socket-python

## Resumo
Projeto para colocar em prática conceitos ensinados no curso de Python e novos como Sockets.

Sistema de pagamentos via PIX com confirmação em tempo real utilizando WebSocket, desenvolvido em Flask.

## Conceitos aplicados
- Flask
- UUID para segurança
- qrcode
- sockets em python
- Gerenciador de pacotes `uv`
- front-end + back-end
- SQLite
- responsabilidades de classe
- Teste unitário com `pytest`
- ORM para modelagem de entidades
- Gerar qrcode com a biblioteca `qrcode`

## Classes e arquivos

### `Payment` — `db_models/payment.py`
Modelo SQLAlchemy que representa um pagamento no banco.

```py
id -> Integer (chave primária)
value -> Float (valor da compra)
paid -> Boolean (se já foi pago, inicia como False)
bank_payment_id -> String (identificador UUID gerado para o PIX)
qr_code -> String (path da imagem do QR Code)
expiration_date -> DateTime (data/hora de criação + 30 min de expiração)
```

**Por quê?** O `bank_payment_id` é um UUID em vez de um inteiro sequencial justamente para evitar que alguém adivinhe IDs de pagamentos alheios.

Possui o método `to_dict()` que converte todos os campos para um dicionário (usado nas respostas JSON da API).

### `Pix` — `payments/pix.py`
Responsável por gerar o código PIX e a imagem do QR Code.

```py
create_payment(base_dir) -> dict
```

Gera:
- `bank_payment_id` → UUID aleatório
- `hash_payment` → string no formato `hash_payment_{uuid}` (simula o copia e cola do PIX)
- `qr_code` → imagem PNG salva em `static/img/qr_code_payment_{uuid}.png`
- Retorna um dict com `bank_payment_id` e `qr_code_path`

**Por quê?** A classe `Pix` isola toda a lógica de geração do PIX em um só lugar, seguindo o princípio de responsabilidade única. Se amanhã o banco mudar o formato do copia e cola, só esse arquivo precisa ser alterado.

### `app.py` — Inicialização e rotas
Arquivo principal que configura Flask, SQLAlchemy e SocketIO, além de definir as rotas da API.

### `repository/database.py`
Apenas instancia o `SQLAlchemy()`. Separado para evitar imports circulares — o `db` é importado tanto pelos models quanto pela app.

## Rotas da API

Usarei o [httpie cli](https://httpie.io/cli) para os exemplos.
O servidor deve estar rodando e o host padrão é `http://127.0.0.1:5000`.

### Criar um pagamento
```bash
http POST http://127.0.0.1:5000/payments/pix value=100.0
```

Resposta:
```json
{
    "message": "The payment has been created",
    "payment": {
        "id": 1,
        "value": 100.0,
        "paid": false,
        "bank_payment_id": "550e8400-e29b-41d4-a716-446655440000",
        "qr-code": "qr_code_payment_550e8400-e29b-41d4-a716-446655440000",
        "expiration_date": "2025-07-01T12:30:00"
    }
}
```

### Obter imagem do QR Code
```bash
http GET http://127.0.0.1:5000/payments/pix/qr_code/qr_code_payment_550e8400-e29b-41d4-a716-446655440000
```
Retorna a imagem PNG do QR Code.

### Confirmar pagamento (simular)
```bash
http POST http://127.0.0.1:5000/payments/pix/confirmation bank_payment_id="550e8400-e29b-41d4-a716-446655440000" value=100.0
```

Resposta:
```json
{
    "message": "The payment has been confirmed"
}
```

**Por quê?** Essa rota simula a notificação que o banco enviaria. Ao confirmar, o servidor emite um evento WebSocket (`payment-confirmed-{id}`) que atualiza a página do comprador em tempo real — sem precisar ficar recarregando.

### Página de acompanhamento
```bash
http GET http://127.0.0.1:5000/payments/pix/1
```
Retorna uma página HTML com o QR Code e o status do pagamento. Se já pago, redireciona para a tela de confirmação.

## Como executar
Primeiro instale as dependências listadas no `pyproject.toml` utilizando `uv` ou `pip`.

```bash
uv sync
```

Após isso, crie um arquivo `.env` na raiz do projeto com o seguinte conteúdo:

```
SECRET_KEY=sua_chave_secreta_aqui
```

Depois execute:

```bash
uv run app.py
```

Acesse `http://127.0.0.1:5000` no navegador.

## Executar testes
```bash
uv run pytest
```

## Estrutura do projeto
```
├── app.py                  # Inicialização do Flask, SocketIO e rotas
├── pyproject.toml          # Configuração do projeto e dependências
├── README.md
│
├── db_models/
│   └── payment.py          # Modelo da entidade Payment (SQLAlchemy)
│
├── instance/
│   └── database.db         # Banco SQLite (gerado automaticamente)
│
├── payments/
│   └── pix.py              # Classe Pix — gera UUID, QR Code e copia e cola
│
├── repository/
│   └── database.py         # Instância do SQLAlchemy (evita imports circulares)
│
├── static/
│   ├── css/
│   │   └── styles.css
│   ├── img/                # QR Codes gerados em execução
│   └── template_img/       # Imagens usadas nos templates
│       ├── 404.png
│       ├── basket.svg
│       ├── check.svg
│       ├── clock.svg
│       ├── currency.svg
│       ├── qrcode.png
│       └── tag.svg
│
├── templates/
│   ├── 404.html
│   ├── confirmed_payment.html
│   └── payment.html        # Tela com QR Code + WebSocket para confirmação em tempo real
│
└── tests/
    └── test_pix.py         # Teste unitário da classe Pix
```
