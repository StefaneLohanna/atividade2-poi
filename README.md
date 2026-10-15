# Atividade 2 — serviços para experimentos de provisionamento

Este repositório contém três backends independentes em FastAPI. Cada serviço
tem uma carga sintética configurável e foi desenhado para ter um recurso
dominante diferente:

- `cpu_service`: recomendação de produtos por similaridade entre vetores;
- `memory_service`: catálogo/cache residente em um buffer grande na memória;
- `io_service`: escrita, leitura e remoção de documentos no disco local.

As aplicações não possuem frontend, autenticação ou banco de dados externo. A
intenção é manter o comportamento controlável e repetível para os experimentos
da disciplina.

## Instalação

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Execução

É possível iniciar cada serviço em um terminal separado:

```bash
uvicorn services.cpu_service.app:app --host 0.0.0.0 --port 8001
uvicorn services.memory_service.app:app --host 0.0.0.0 --port 8002
uvicorn services.io_service.app:app --host 0.0.0.0 --port 8003
```

Também é possível executar diretamente os módulos:

```bash
python services/cpu_service/app.py
python services/memory_service/app.py
python services/io_service/app.py
```

Cada serviço disponibiliza documentação interativa em `/docs` e um endpoint
`/health`.

## Exemplos de uso

### CPU-bound — recomendação

O catálogo é gerado deterministicamente pelo `seed` e fica em um cache limitado
para que as requisições seguintes meçam principalmente o cálculo de
similaridade.

```bash
curl -X POST http://localhost:8001/recommend \
  -H 'content-type: application/json' \
  -d '{"items": 10000, "dimensions": 64, "top_k": 10, "iterations": 2, "seed": 42}'
```

Os principais controles são `items`, `dimensions` e `iterations`. O número de
operações aproximado é `items * dimensions * iterations`.

### Memory-bound — catálogo em memória

O serviço aloca um único conjunto de trabalho residente. Alterar `dataset_mb`
substitui o conjunto anterior; repetir a mesma configuração aproveita o buffer
já carregado.

```bash
curl 'http://localhost:8002/cache/scan?dataset_mb=128&passes=3&mode=read&seed=42'
curl 'http://localhost:8002/cache/status'
```

`mode=read_write` também grava em cada passagem:

```bash
curl 'http://localhost:8002/cache/scan?dataset_mb=256&passes=2&mode=read_write'
```

### IO-bound — documentos

Para um teste que grava, lê e remove arquivos temporários:

```bash
curl -X POST \
  'http://localhost:8003/documents/workload?size_mb=32&repetitions=2&sync=false'
```

Para manter um documento no diretório de dados e depois consultá-lo:

```bash
created=$(curl -s -X POST 'http://localhost:8003/documents?size_mb=8&sync=true')
echo "$created"
curl http://localhost:8003/documents/<id-retornado>
curl -X DELETE http://localhost:8003/documents/<id-retornado>
```

O diretório pode ser alterado sem modificar o código:

```bash
IO_DATA_DIR=/local/uasc/poi-io-data \
  uvicorn services.io_service.app:app --host 0.0.0.0 --port 8003
```

`sync=true` chama `fsync` depois da escrita e é útil para comparar uma escrita
confirmada no dispositivo com uma escrita que pode permanecer no cache do
sistema operacional.

## Testes

```bash
pytest -q
```

Os testes são smoke tests: validam o contrato básico dos três endpoints sem
executar cargas grandes.

## Experimento de provisionamento

O serviço previsto para a VM é o de memória. A demanda é o tamanho do conjunto
de trabalho; o provisionamento (vCPU e RAM) é o que a VirtualBox estiver
oferecendo no momento da coleta. O repositório não altera a VM. Os quatro
cenários abaixo são rótulos para repetir a mesma demanda em máquinas diferentes:

| Cenário | vCPU | RAM |
| --- | --- | --- |
| `1vcpu-1gb` | 1 | 1 GiB |
| `1vcpu-2gb` | 1 | 2 GiB |
| `2vcpu-2gb` | 2 | 2 GiB |
| `2vcpu-4gb` | 2 | 4 GiB |

Em cada cenário as demandas são 64, 256, 512 e 1024 MiB, com 4 requisições
simultâneas durante 20 segundos. Dois pedidos de aquecimento ficam de fora da
conta, para a medição não incluir a alocação do buffer. O modo é `read`: várias
requisições varrem o mesmo buffer sem alterá-lo.

A comparação que interessa: com 1 GiB, 512 e 1024 MiB devem degradar (swap ou
falha por falta de memória); com 4 GiB a mesma demanda deve continuar estável.
O par `1vcpu-2gb` e `2vcpu-2gb` mostra se um núcleo a mais muda a vazão quando
a RAM é a mesma. Confira se a VM tem swap (`swapon --show`). Sem swap, a demanda
que não cabe mata o processo em vez de só aumentar o tempo de resposta.

Na VM, depois de ajustar CPU e RAM e reiniciar:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
./scripts/start_memory.sh
```

Em outro terminal, com o virtualenv ativo, um cenário por vez:

```bash
python3 experiments/run_load.py --scenario 1vcpu-1gb
```

Repita com `1vcpu-2gb`, `2vcpu-2gb` e `2vcpu-4gb`, sempre depois de desligar a
VM, mudar o provisionamento e subir de novo. Cada execução regrava
`experiments/results/memory_<cenário>.csv` com tempo de resposta (mediana e
p95, medidos no cliente), vazão, CPU do sistema, iowait e memória residente do
processo. O RSS inclui o interpretador, não só o buffer: compare níveis e
cenários entre si. A diferença de RSS entre duas demandas deve acompanhar a
diferença de tamanho do dataset.

Os gráficos saem na máquina em que os CSV estiverem reunidos. O Matplotlib fica
fora do deploy da VM:

```bash
pip install -r requirements-experiments.txt
python3 experiments/plot_results.py
```

Isso gera `experiments/plots/memory_resumo.png`: tempo de resposta, vazão, CPU
e memória, uma curva por cenário. A linha tracejada no tempo de resposta é o
p95. Resultados e figuras estão no `.gitignore`; o documento da atividade é
quem leva os gráficos. Se os níveis menores parecerem limitados pelo HTTP e não
pela memória, aumente `--passes` com o mesmo valor em todos os cenários.

Os outros dois serviços usam o mesmo coletor, se a VM ficar com eles em vez do
de memória:

```bash
./scripts/start_cpu.sh
python3 experiments/run_load.py --service cpu --scenario 1vcpu-1gb --demands 2000,5000,10000,20000

./scripts/start_io.sh
python3 experiments/run_load.py --service io --scenario 1vcpu-1gb --demands 8,32,128
```

`start_cpu.sh` abre um worker por vCPU, porque o cálculo em Python não usa mais
do que um núcleo por processo. `start_io.sh` grava em `.data/io` (disco do
projeto, não `/tmp`) e a carga usa `fsync`.

