# Sistema Automatizado de Vigilância de Fronteira
## Pseudocódigo Completo — AV Final IAC

```
DEFINIR VALOR_SUSPEITO = R$ 10.000,00
DEFINIR JANELA_HORAS = 72
DEFINIR PRECISAO_FACIAL = 95%
DEFINIR PRECISAO_PLACA = 90%
DEFINIR TIMEOUT_GATEWAY = 5s
DEFINIR BUFFER_OFFLINE_MAX = 500 registros


-- Leitura de placa

FUNÇÃO lerPlaca(foto):
  placa <- IA.lerPlaca(foto)
  precisao <- IA.precisao()
  SE precisao >= PRECISAO_PLACA ENTÃO
    RETORNAR placa
  SENÃO
    REGISTRAR erro('placa: precisão insuficiente')
    RETORNAR NULO
  FIM SE
FIM FUNÇÃO


-- Reconhecimento facial

FUNÇÃO identificarPessoa(foto):
  rosto <- IA.detectarRosto(foto)
  precisao <- IA.precisao()
  SE precisao >= PRECISAO_FACIAL ENTÃO
    pessoa <- BD.buscarPorRosto(rosto)
    RETORNAR pessoa
  SENÃO
    REGISTRAR erro('facial: precisão insuficiente')
    RETORNAR NULO
  FIM SE
FIM FUNÇÃO


-- Consulta ao banco de dados (com validação de schema)

FUNÇÃO validarSchema(registro, camposEsperados):
  PARA CADA campo EM camposEsperados FAÇA
    SE registro[campo] == NULO OU TIPO(registro[campo]) != campo.tipoEsperado ENTÃO
      RETORNAR FALSO
    FIM SE
  FIM PARA
  RETORNAR VERDADEIRO
FIM FUNÇÃO

FUNÇÃO buscarDados(placa, pessoa):
  veiculo   <- BD.buscar(CRIPTOGRAFAR(placa))
  cadastro  <- BD.buscar(CRIPTOGRAFAR(pessoa.CPF))
  historico <- BD.buscarViagens(pessoa.CPF)

  SE veiculo == NULO OU cadastro == NULO ENTÃO
    RETORNAR NULO
  FIM SE

  SE NÃO validarSchema(veiculo,  ['placa','modelo','cor']) OU
     NÃO validarSchema(cadastro, ['CPF','nome','dataNasc']) ENTÃO
    REGISTRAR erro('schema inválido: dado descartado')
    RETORNAR NULO
  FIM SE

  RETORNAR { veiculo, cadastro, historico }
FIM FUNÇÃO


-- Análise financeira

FUNÇÃO analisarBanco(CPF, dataViagem):
  transacoes <- BANCO.buscar(CPF, dataViagem - 72h, dataViagem + 72h)
  alertas <- []

  PARA CADA tx EM transacoes FAÇA
    SE tx.valor >= VALOR_SUSPEITO ENTÃO
      alertas.adicionar('VALOR_ALTO')
    FIM SE
    SE saques24h(CPF) >= 5 ENTÃO
      alertas.adicionar('FRACIONAMENTO')
    FIM SE
    SE tx.tipo == 'INTERNACIONAL' ENTÃO
      alertas.adicionar('INTERNACIONAL')
    FIM SE
  FIM PARA

  RETORNAR alertas
FIM FUNÇÃO


-- Cálculo de risco

FUNÇÃO calcularRisco(alertas, historico):
  pontos <- 0
  SE 'FRACIONAMENTO' EM alertas ENTÃO pontos <- pontos + 40
  SE 'VALOR_ALTO'    EM alertas ENTÃO pontos <- pontos + 30
  SE 'INTERNACIONAL' EM alertas ENTÃO pontos <- pontos + 20
  SE historico.viagensMes >= 10  ENTÃO pontos <- pontos + 20

  SE pontos >= 70 ENTÃO RETORNAR 'ALTO'
  SE pontos >= 40 ENTÃO RETORNAR 'MEDIO'
  CASO CONTRARIO       RETORNAR 'BAIXO'
FIM FUNÇÃO


-- Alerta à PRF (com transporte seguro)

FUNÇÃO alertarPRF(pessoa, veiculo, alertas, risco):
  SE risco == 'ALTO' OU risco == 'MEDIO' ENTÃO
    conexao <- REDE.abrirConexao(servidorPRF, protocolo='HTTPS', tls='1.3')

    SE conexao == NULO ENTÃO
      REGISTRAR erro('falha ao abrir conexão segura com a PRF')
      bufferOffline.adicionar({ pessoa, veiculo, alertas, risco })
      RETORNAR
    FIM SE

    PRF.enviar(conexao, {
      pessoa:  CRIPTOGRAFAR(pessoa),
      veiculo: CRIPTOGRAFAR(veiculo),
      alertas: CRIPTOGRAFAR(alertas),
      acao:    'INTERCEPTAR'
    })
  SENÃO
    REGISTRAR log('Risco baixo, sem acao')
  FIM SE
FIM FUNÇÃO


-- Loop principal (com contingência de energia)

INÍCIO
  ENQUANTO sistema.ligado FAÇA

    SE energia.status == 'FALHA' ENTÃO
      energia.acionarBackup()
    FIM SE

    placa  <- lerPlaca(câmera.frente())
    pessoa <- identificarPessoa(câmera.interna())

    SE placa == NULO OU pessoa == NULO ENTÃO CONTINUAR FIM SE

    dados <- buscarDados(placa, pessoa)
    SE dados == NULO ENTÃO CONTINUAR FIM SE

    alertas <- analisarBanco(dados.CPF, agora())
    risco   <- calcularRisco(alertas, dados.historico)

    alertarPRF(dados.pessoa, dados.veiculo, alertas, risco)

    SE rede.status == 'OFFLINE' ENTÃO
      bufferOffline.salvar(placa, pessoa.CPF, risco)
    SENÃO
      REGISTRAR auditoria(placa, pessoa.CPF, risco)
      bufferOffline.sincronizarPendentes()
    FIM SE

  FIM ENQUANTO
FIM
```
