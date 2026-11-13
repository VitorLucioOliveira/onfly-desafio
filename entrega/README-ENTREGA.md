# Entrega — Consolidador de reservas de hotel

## Investigação inicial

Comecei executando os testes fornecidos, sem modificá-los, para identificar os comportamentos que não atendiam às regras do `README.md`. A execução inicial apresentou cinco falhas:

```text
FAILED tests/test_currency.py::test_currency_conversion_to_brl
FAILED tests/test_nights.py::test_nights_across_month_boundary
FAILED tests/test_regression_report.py::test_regression_report_on_sample
FAILED tests/test_status_rules.py::test_status_rules_exclude_cancelled_and_no_show
FAILED tests/test_top_hotels.py::test_top_hotels_are_ranked_descending
5 failed in 0.23s
```

A partir desses resultados, investiguei cada cenário comparando o valor obtido com o esperado. Usei as regras de negócio como referência para levantar hipóteses, localizar a função responsável e verificar o efeito de cada correção nos testes.

## Defeitos corrigidos e abordagem

### 1. Conversão de moedas para BRL

**Teste:** `tests/test_currency.py::test_currency_conversion_to_brl`

```text
AssertionError: assert '118.45' == '642.00'
```

**Causa raiz:** a conversão dividia o valor pela taxa de câmbio, embora a regra exigisse multiplicar `amount` por `rate_to_brl`.

Como o teste tratava de conversão cambial, minha primeira hipótese foi um erro na operação usada para converter os valores. Ao verificar a função, encontrei:

```python
return Decimal(str(float(amount) / float(rate)))
```

Troquei a divisão pela multiplicação e compri o requisito de somente usar valores `Decimal`. Enfim executei os testes novamente:

```python
return amount * rate
```

Com essa alteração, o teste passou a retornar o valor esperado de `"642.00"`.



### 2. Cálculo de noites na virada de mês

**Teste:** `tests/test_nights.py::test_nights_across_month_boundary`

```text
assert -28 == 3
```

**Causa raiz:** o cálculo subtraía apenas o dia do mês de cada data, desconsiderando o mês e o ano da estadia.

O teste usava check-in em `2025-01-30` e check-out em `2025-02-02`, com um quarto. O resultado esperado era de três noites, mas a função retornava `-28`. Esse valor sugeria que o cálculo estava fazendo apenas `2 - 30`, o que confirmei ao analisar a implementação:

```python
return (checkout.day - checkin.day) * rooms
```

Substituí a subtração dos dias isolados pela diferença entre as datas completas:

```python
return (checkout - checkin).days * rooms
```

Assim, a função considera corretamente a passagem de mês ou ano e multiplica a duração da estadia pela quantidade de quartos. A diferença permanece não inclusiva, conforme a regra do enunciado.

### 3. Inclusão indevida de reservas com status `NO_SHOW`

**Teste:** `tests/test_status_rules.py::test_status_rules_exclude_cancelled_and_no_show`

```text
AssertionError: assert '120.00' == '100.00'
```

**Causa raiz:** o filtro excluía somente reservas `CANCELLED`, permitindo que reservas `NO_SHOW` também entrassem nas agregações.

O cenário do teste continha estas reservas:

| Reserva | Status | Valor |
| --- | --- | ---: |
| R-1 | `CONFIRMED` | 100.00 |
| R-2 | `CANCELLED` | 50.00 |
| R-3 | `NO_SHOW` | 20.00 |

Como o total obtido era `120.00`, levantei a hipótese de que o valor da reserva `NO_SHOW` estava sendo somado ao da reserva confirmada: `100 + 20 = 120`.

Ao verificar a função que define se uma reserva é faturável, encontrei:

```python
return status != "CANCELLED"
```

Ajustei a condição para aceitar exclusivamente o status previsto na regra de negócio:

```python
return status == "CONFIRMED"
```

Com isso, apenas reservas confirmadas entram em noites, valores, `by_month` e `top_hotels`. As reservas canceladas e os casos de não comparecimento continuam em `total_reservations` e nas respectivas contagens por status, pois essa contagem ocorre antes do filtro.

### 4. Ordenação do ranking de hotéis

**Teste:** `tests/test_top_hotels.py::test_top_hotels_are_ranked_descending`

```text
AssertionError: assert ['H04', 'H03', 'H02'] == ['H01', 'H02', 'H03']
```

**Causa raiz:** o ranking ordenava os hotéis pelo valor total em ordem crescente, enquanto o contrato exigia ordem decrescente, com desempate por `hotel_id` crescente.

Ao investigar a montagem do ranking, encontrei:

```python
ranked_hotels = sorted(
    hotels.items(),
    key=lambda item: (item[1]["amount"], item[0]),
)
```

O `sorted()` usa ordem crescente por padrão. Como a chave é uma tupla, ele compara primeiro o valor de `amount` e, em caso de empate, o identificador do hotel.

Considerei inverter a ordenação ou usar o valor negativo no primeiro critério. Escolhi a segunda opção para preservar a estrutura original e manter o desempate por identificador em ordem crescente:

```python
ranked_hotels = sorted(
    hotels.items(),
    key=lambda item: (-item[1]["amount"], item[0]),
)
```

Dessa forma, os hotéis com maior total aparecem primeiro. Usar `reverse=True` nessa mesma chave inverteria também a ordem dos identificadores nos empates, contrariando o requisito.

### 5. Falha no teste de regressão do relatório

**Teste:** `tests/test_regression_report.py::test_regression_report_on_sample`

```text
AssertionError: assert {'average_nig...ions': 0, ...} == {'average_nig...ions': 0, ...}
```

**Causa raiz identificada na investigação:** a conversão cambial incorreta também afetava os valores consolidados comparados pelo teste de regressão.

Esse teste passou após a correção da operação de câmbio descrita no item 1, sem exigir uma alteração específica no teste ou na estrutura do relatório. Isso indicou que a falha era um reflexo do mesmo defeito na conversão, e não necessariamente um quinto problema independente.

## Como validar

Execute os comandos abaixo a partir da raiz do projeto, usando Python 3.11 ou superior.

### Preparar o ambiente

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
```

### Executar os testes

```bash
pytest -q
```

Na verificação desta revisão, com Python 3.11.18, o resultado foi:

```text
5 passed in 0.22s
```

### Gerar o relatório completo

```bash
PYTHONPATH=src python -m onfly_report \
  --input data/reservas_hoteis.csv \
  --rates data/taxas_cambio.csv \
  --output output/report.json \
  --top 5
```

A execução de validação terminou com sucesso e apresentou o seguinte resumo:

```text
Reservas: 200 | Confirmadas: 153 | Noites: 909 | Total BRL: 249543.79
```

### Conferir a amostra do enunciado

```bash
PYTHONPATH=src python -m onfly_report \
  --input data/exemplo-validacao.csv \
  --rates data/taxas_cambio.csv \
  --output output/exemplo.json
```

Na validação, os valores do resumo, os agrupamentos mensais e o ranking coincidiram com a amostra do `README.md`. Os principais totais foram:

| Campo | Valor obtido e esperado |
| --- | ---: |
| `total_reservations` | 12 |
| `confirmed_reservations` | 9 |
| `cancelled_reservations` | 2 |
| `no_show_reservations` | 1 |
| `total_room_nights` | 28 |
| `total_amount_brl` | `"7566.00"` |
| `average_nightly_rate_brl` | `"270.21"` |

As execuções desta revisão gravaram os relatórios em arquivos temporários para preservar os arquivos existentes em `output/`. Os comandos acima usam os caminhos de entrega previstos no enunciado.

## Checklist

- [x] O CLI roda com Python 3.11 (`PYTHONPATH=src python -m onfly\_report ...`).
- [x] O comando de execução gera `output/report.json` sem erro.
- [x] `pytest -q` passa com os testes fornecidos.
- [x] `entrega/README-ENTREGA.md` preenchido.
- [x] Sessão + declaração apenas no caminho com IA; investigação no README-ENTREGA nos dois caminhos.
- [x] Repositório público no GitHub com o projeto e o link enviado.


