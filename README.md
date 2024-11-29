# Projeto_Final_IoT

# Documentação

# * Codigo Python

from flask import render_template # Importa a função para renderizar templates HTML
from app import app # Importa a instância do aplicativo Flask
import RPi.GPIO as gpio # Biblioteca para controle de GPIO no Raspberry Pi
import time as delay # Importa a biblioteca time e a renomeia como "delay"
from datetime import datetime # Importa a classe para manipulação de datas e horas
from urllib.request import urlopen # Importa função para realizar requisições HTTP
import requests # Biblioteca para realizar requisições HTTP

### Define pinos GPIO para LEDs e sensores
ledVermelho, ledVerde = 11, 12
pin_e = 16 # Pino do Echo (sensor ultrassônico)
pin_t = 15 # Pino do Trigger (sensor ultrassônico)

### Configura URLs e chaves para integração com ThingSpeak
urlBase = 'https://api.thingspeak.com/update?api_key='
keyWrite = 'PZTZJH6WNKIFWOME' # Chave para envio de dados
sensorDistancia = '&field1='
urlBase_read = 'https://api.thingspeak.com/channels/'
readKey = '/last?key=3BE8TI8EO588TTUP' # Chave para leitura de dados
channels = '2746094' # Canal ThingSpeak

### Configuração de campos do ThingSpeak
field1 = '/fields/1/'
field2 = '/fields/2/'

### Variáveis globais
ocupacao = 0 # Nível de ocupação inicial da lixeira
lixeira_v = 20 # Altura da lixeira em centímetros
lista_registro = [] # Lista para armazenar registros de ações
status_tampa = True # Estado inicial da tampa (aberta)

### Configuração inicial do GPIO
gpio.setmode(gpio.BOARD) # Define o modo de numeração dos pinos como BOARD
gpio.setwarnings(False) # Desativa os avisos do GPIO
gpio.setup(ledVermelho, gpio.OUT) # Configura o pino do LED vermelho como saída
gpio.setup(ledVerde, gpio.OUT) # Configura o pino do LED verde como saída
gpio.output(ledVermelho, gpio.LOW) # Inicializa o LED vermelho como desligado
gpio.output(ledVerde, gpio.LOW) # Inicializa o LED verde como desligado

### Configuração dos sensores ultrassônicos
gpio.setup(pin_t, gpio.OUT) # Configura o Trigger como saída
gpio.setup(pin_e, gpio.IN) # Configura o Echo como entrada

### Função para abrir a tampa da lixeira
def abre_tampa():
    i = 0
    while i <= 3: # Pisca o LED verde 3 vezes
        gpio.output(ledVerde, gpio.HIGH) # Liga o LED verde
        delay.sleep(1) # Espera 1 segundo
        gpio.output(ledVerde, gpio.LOW) # Desliga o LED verde
        delay.sleep(1) # Espera 1 segundo
        i += 1
    return True # Retorna True indicando que a tampa está aberta

### Função para fechar a tampa da lixeira
def fecha_tampa():
    i = 0
    while i <= 3: # Pisca o LED vermelho 3 vezes
        gpio.output(ledVermelho, gpio.HIGH) # Liga o LED vermelho
        delay.sleep(1) # Espera 1 segundo
        gpio.output(ledVermelho, gpio.LOW) # Desliga o LED vermelho
        delay.sleep(1) # Espera 1 segundo
        i += 1
    return False # Retorna False indicando que a tampa está fechada

### Função para verificar o status da lixeira
def status_lixeira():
    disponivel = True # Inicialmente, a lixeira está disponível
    while True: # Loop contínuo para monitoramento
        if ocupacao < 100: # Se a ocupação for menor que 100%
            gpio.output(ledVerde, gpio.HIGH) # Liga o LED verde
            disponivel = True
        else: # Se a ocupação for maior ou igual a 100%
            gpio.output(ledVermelho, gpio.HIGH) # Liga o LED vermelho
            disponivel = False
        return disponivel # Retorna o status da lixeira

### Função para obter a última atualização
def ultima_atualizacao():
    now = datetime.now() # Obtém a data e hora atuais
    time = now.strftime("%d/%m/%Y %H:%M") # Formata no padrão DD/MM/AAAA HH:MM
    return time # Retorna a data e hora formatadas

### Função para registrar as ações da tampa (abrir/fechar)
def resgitro_tampa(status):
    now = datetime.now() # Obtém a data e hora atuais
    time = now.strftime("%d/%m/%Y %H:%M") # Formata a data e hora
    if status == True: # Se a tampa foi aberta
        lista_registro.append(('Abriu Tampa', time))
    else: # Se a tampa foi fechada
        lista_registro.append(('Fechou Tampa', time))
    return lista_registro # Retorna a lista atualizada de registros

### Função para testar conexão com a internet
def testaConexao():
    try:
        urlopen('https://www.materdei.edu.br/pt', timeout=1) # Faz um ping em um site
        return True # Retorna True se houver conexão
    except:
        return False # Retorna False se não houver conexão

### Função para calcular a distância medida pelo sensor ultrassônico
def distancia():
    gpio.output(pin_t, True) # Envia um pulso no Trigger
    delay.sleep(0.000001) # Pulso curto de 1 microsegundo
    gpio.output(pin_t, False) # Para o pulso
    tempo_i = delay.time() # Marca o início do tempo
    tempo_f = delay.time() # Marca o fim do tempo
    while gpio.input(pin_e) == False: # Aguarda o retorno do sinal
        tempo_i = delay.time()
    while gpio.input(pin_e) == True: # Calcula o tempo de retorno
        tempo_f = delay.time()
    temp_d = tempo_f - tempo_i # Tempo de ida e volta do sinal
    distancia = (temp_d * 34300) / 2 # Converte o tempo em distância

    ocupacao_l = (distancia / lixeira_v) * 100 # Calcula a ocupação em %
    if ocupacao_l <= 2: # Se a ocupação for muito baixa
        ocupacao_lixeira = ('{0:0.0f}%'.format(100))
    else: # Calcula a ocupação restante
        ocupacao_f = 100 - ocupacao_l
        ocupacao_lixeira = ('{0:0.0f}%'.format(ocupacao_f))

    return ocupacao_lixeira # Retorna a ocupação formatada

### Função para enviar dados ao ThingSpeak
def envia_dados():
    if testaConexao() == True: # Testa conexão com a internet
        urlDados = (urlBase + keyWrite + sensorDistancia + str(distancia())) # Monta a URL com os dados
        retorno = requests.post(urlDados) # Envia uma requisição POST
        if retorno.status_code == 200: # Se o código de resposta for 200
            print('Dados enviados com sucesso')
        else:
            print('Erro ao enviar dados: ' + str(retorno.status_code))
    else:
        print('Sem conexão') # Sem conexão com a internet

### Função para consultar dados do ThingSpeak
def consulta_dados():
    if testaConexao() == True: # Testa conexão com a internet
        print('Conexão OK')
        while True: # Loop contínuo
            consultaDistancia = requests.get(urlBase + channels + field1 + readKey) # Faz uma requisição GET
            print(consultaDistancia.text) # Exibe os dados obtidos
            delay.sleep(20) # Aguarda 20 segundos antes de repetir
    else:
        print('Sem conexão com a INTERNET')

### Rota principal do Flask
@app.route('/')
def index():
    templateData = {
        'status_lixeira': status_lixeira(), # Obtém o status da lixeira
        'ocup_lixeira': distancia(), # Calcula a ocupação
        'registro_tampa': resgitro_tampa(status_tampa), # Registra as ações da tampa
        'atualizacao': ultima_atualizacao() # Obtém a última atualização
    }
    return render_template('index.html', **templateData) # Renderiza o template HTML com os dados

### Rota para abrir a tampa
@app.route('/abre-tampa')
def abrir_tampa():
    status_tampa = abre_tampa() # Abre a tampa
    envia_dados() # Envia os dados atualizados
    templateData = {
        'status_lixeira': status_lixeira(),
        'ocup_lixeira': distancia(),
        'registro_tampa': resgitro_tampa(status_tampa),
        'atualizacao': ultima_atualizacao()
    }
    return render_template('index.html', **templateData) # Renderiza o template atualizado

### Rota para fechar a tampa
@app.route('/fechar-tampa')
def fechar_tampa():
    status_tampa = fecha_tampa() # Fecha a tampa
    envia_dados() # En

# * HTML

Aqui está o código HTML documentado com comentários linha a linha:

```html
<!DOCTYPE html>
<html lang="pt-br"> <!-- Define o documento como HTML5 e define o idioma como português do Brasil -->
<head>
    <meta charset="UTF-8"> <!-- Define a codificação de caracteres como UTF-8 -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0"> <!-- Torna o site responsivo em diferentes tamanhos de tela -->
    <link rel="stylesheet" href="../static/bootstrap.min.css"> <!-- Importa o CSS do Bootstrap para estilização -->
    <link rel="stylesheet" href="../static/css/style.css"> <!-- Importa o CSS personalizado do projeto -->
    <title>Lixeira Inteligente</title> <!-- Define o título da página -->
</head>
<body>
    <!-- Título principal da página -->
    <div class="title">
        <h1>Lixeira Inteligente</h1> <!-- Cabeçalho que exibe o título do projeto -->
    </div>

    <!-- Seção de controles para abrir e fechar a tampa da lixeira -->
    <section class="controles">
        <h2>Controles</h2> <!-- Subtítulo da seção -->
        <a class="button_fechar_tampa" href="/fechar-tampa">Fechar Tampa</a> <!-- Botão para fechar a tampa (redireciona para o endpoint) -->
        <a class="button_abrir_tampa" href="/abre-tampa">Abrir Tampa</a> <!-- Botão para abrir a tampa (redireciona para o endpoint) -->
    </section>

    <!-- Seção que exibe o status atual da lixeira -->
    <section class="status_lixeira">
        <h2>Status da Lixeira</h2> <!-- Subtítulo da seção -->
        <div class="license-status"> <!-- Container para status e detalhes -->
            <div class="status-label">Status da Lixeira</div> <!-- Rótulo do status -->
            <div class="status-value">
                {% if status_lixeira == True %} <!-- Verifica se a variável 'status_lixeira' é True -->
                    Disponível <!-- Exibe 'Disponível' se a condição for verdadeira -->
                {% else %}
                    Indisponível <!-- Exibe 'Indisponível' se a condição for falsa -->
                {% endif %}
            </div>
            <div class="details-label">Detalhes</div> <!-- Rótulo dos detalhes -->
            <div class="details-value">Última atualização: {{atualizacao}}</div> <!-- Exibe a última atualização usando a variável 'atualizacao' -->
        </div>
    </section>

    <!-- Seção para monitoramento em tempo real da ocupação da lixeira -->
    <section class="monitoramento">
        <h2>Monitoramento em tempo Real</h2> <!-- Subtítulo da seção -->
        <div class="real-time-monitoring"> <!-- Container para o monitoramento -->
            <div class="occupation-label">Ocupação</div> <!-- Rótulo para a ocupação -->
            <div class="occupation-bar" style="width: {{ocup_lixeira}}"></div> <!-- Barra de progresso indicando a ocupação -->
            <h5>{{ocup_lixeira}} de ocupação</h5> <!-- Exibe o nível de ocupação como texto -->
        </div>
    </section>

    <!-- Seção que apresenta o histórico de ações realizadas na lixeira -->
    <section class="historico">
        <h2>Historico</h2> <!-- Subtítulo da seção -->
        <table> <!-- Início da tabela -->
            <tr> <!-- Linha de cabeçalho da tabela -->
                <th>Data</th> <!-- Cabeçalho para a coluna de datas -->
                <th>Evento</th> <!-- Cabeçalho para a coluna de eventos -->
            </tr>
            {% for acao, horario in registro_tampa %} <!-- Itera sobre a lista 'registro_tampa', que contém ações e horários -->
                <tr> <!-- Cria uma linha para cada registro -->
                    <td>{{ acao }}</td> <!-- Exibe a ação realizada -->
                    <td>{{ horario }}</td> <!-- Exibe o horário da ação -->
                </tr>
            {% endfor %} <!-- Fim do loop -->
        </table>
    </section>

</body>
</html>
```