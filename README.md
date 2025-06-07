;***************************************************************************
;* TRABALHO ASSEMBLY - ATMEGA2560                                          *
;* Processamento de caracteres com tabela de referência.                   *
;***************************************************************************

;===========================================================================
; DIRETIVAS E DEFINIÇÕES
;===========================================================================
.INCLUDE "m2560def.inc"

;-- Definição de registradores para melhor legibilidade
.def temp           = r16  ; Registrador de uso geral
.def temp2          = r17  ; Registrador de uso geral 2
.def char_atual     = r18  ; Armazena o caractere sendo processado
.def contador       = r19  ; Conta as ocorrências do caractere
.def flag_encontrado= r20  ; Flag (0 ou 1) se o caractere está na tabela de referência

;-- Definição dos ponteiros
; X (r27:r26) -> Usado para apontar para a Tabela de Saída na RAM
; Y (r29:r28) -> Usado para apontar para a Frase de Teste na RAM
; Z (r31:r30) -> Usado para ler da Flash e para laços internos na RAM

;-- Definição de constantes de endereço na RAM
.equ LOOKUP_TBL_ADDR = 0x0200  ; Endereço inicial da tabela de referência na RAM
.equ PHRASE_ADDR     = 0x0300  ; Endereço inicial da frase na RAM
.equ OUTPUT_TBL_ADDR = 0x0400  ; Endereço inicial da tabela de saída na RAM

;===========================================================================
; SEGMENTO DE DADOS (ARMAZENADOS NA FLASH PARA COPIAR PARA A RAM)
;===========================================================================
.CSEG
.ORG 0x100 ; Coloca os dados em um local seguro na memória de programa

;-- Tabela de Referência (15 caracteres alfanuméricos)
LOOKUP_TABLE_FLASH:
.DB 'A', 'E', 'I', 'O', 'U', 's', 't', 'm', 'g', 'z', '0', '2', '5', '6', '9'
.equ LOOKUP_TABLE_SIZE = 15

;-- Frase de Teste (Nome e 4 dígitos da matrícula)
;-- Termina com 0x00 (NULO) para marcar o fim da string
PHRASE_FLASH:
.DB "Lucas Alves 2024"
.DB 0x00
.equ PHRASE_SIZE = 16 ; Incluindo o NULO

;===========================================================================
; SEGMENTO DE CÓDIGO
;===========================================================================
.CSEG
.ORG 0x0000
    rjmp MAIN       ; Pula para o início do programa principal

;---------------------------------------------------------------------------
; ROTINA PRINCIPAL
;---------------------------------------------------------------------------
MAIN:
    ;-- 1. Inicializa o Stack Pointer (essencial para CALL/RET)
    ldi temp, low(RAMEND)
    out SPL, temp
    ldi temp, high(RAMEND)
    out SPH, temp

    ;-- 2. Copia os dados da Flash para a RAM
    ; Copia a Tabela de Referência
    ldi ZL, low(LOOKUP_TABLE_FLASH*2) ; Z aponta para a fonte na Flash
    ldi ZH, high(LOOKUP_TABLE_FLASH*2)
    ldi XL, low(LOOKUP_TBL_ADDR)      ; X aponta para o destino na RAM
    ldi XH, high(LOOKUP_TBL_ADDR)
    ldi temp, LOOKUP_TABLE_SIZE       ; Número de bytes a copiar
    call COPY_FLASH_TO_RAM

    ; Copia a Frase de Teste
    ldi ZL, low(PHRASE_FLASH*2)       ; Z aponta para a fonte na Flash
    ldi ZH, high(PHRASE_FLASH*2)
    ldi XL, low(PHRASE_ADDR)          ; X aponta para o destino na RAM
    ldi XH, high(PHRASE_ADDR)
    ldi temp, PHRASE_SIZE             ; Número de bytes a copiar
    call COPY_FLASH_TO_RAM

    ;-- 3. Inicia o processamento da frase
    rcall PROCESS_PHRASE

DONE:
    rjmp DONE       ; Loop infinito para "parar" o processador

;---------------------------------------------------------------------------
; Sub-rotina: Processa a frase e cria a tabela de saída
;---------------------------------------------------------------------------
PROCESS_PHRASE:
    ; Inicializa ponteiros para as tabelas na RAM
    ldi YL, low(PHRASE_ADDR)      ; Y aponta para o início da frase
    ldi YH, high(PHRASE_ADDR)
    ldi XL, low(OUTPUT_TBL_ADDR)  ; X aponta para o início da tabela de saída
    ldi XH, high(OUTPUT_TBL_ADDR)

PROCESS_LOOP:
    ; Carrega o caractere atual da frase (apontado por Y)
    ld char_atual, Y
    cpi char_atual, 0x00          ; Verifica se é o final da frase
    breq PROCESS_END              ; Se for, termina o processamento

    ;-- Antes de processar, verifica se o caractere já está na tabela de saída
    push YL                       ; Salva o ponteiro Y
    push YH
    ldi ZL, low(OUTPUT_TBL_ADDR)  ; Z aponta para o início da tabela de saída
    ldi ZH, high(OUTPUT_TBL_ADDR)
CHECK_IF_PROCESSED_LOOP:
    cp ZL, XL                     ; Compara Z com X. Se iguais, chegamos ao fim da tabela de saída
    cpc ZH, XH
    breq NOT_PROCESSED_YET        ; Se não foi processado ainda, pula para o processamento
    ld temp, Z                    ; Carrega o caractere da tabela de saída
    cp temp, char_atual           ; Compara com o caractere atual
    breq ALREADY_PROCESSED        ; Se for igual, já foi processado, então pega o próximo da frase
    adiw Z, 3                     ; Avança para a próxima entrada na tabela de saída (char, count, flag)
    rjmp CHECK_IF_PROCESSED_LOOP

ALREADY_PROCESSED:
    pop YH                        ; Restaura o ponteiro Y
    pop YL
    adiw Y, 1                     ; Pula para o próximo caractere da frase
    rjmp PROCESS_LOOP             ; E reinicia o loop

NOT_PROCESSED_YET:
    pop YH                        ; Restaura o ponteiro Y
    pop YL

    ;-- CONTAGEM DE OCORRÊNCIAS --
    ldi contador, 0               ; Zera o contador
    push YL                       ; Salva o ponteiro Y
    push YH
    ldi ZL, low(PHRASE_ADDR)      ; Z percorrerá a frase para contar
    ldi ZH, high(PHRASE_ADDR)
COUNT_LOOP:
    ld temp, Z                    ; Carrega caractere da frase com o ponteiro Z
    cpi temp, 0x00                ; Fim da frase?
    breq COUNT_END                ; Se sim, termina a contagem
    cp temp, char_atual           ; Compara com o caractere que estamos analisando
    brne COUNT_NEXT               ; Se não for igual, pula
    inc contador                  ; Se for igual, incrementa o contador
COUNT_NEXT:
    adiw Z, 1                     ; Próximo caractere
    rjmp COUNT_LOOP
COUNT_END:
    pop YH                        ; Restaura Y
    pop YL

    ;-- VERIFICAÇÃO NA TABELA DE REFERÊNCIA --
    ldi flag_encontrado, 0        ; Zera a flag (assume que não pertence)
    ldi temp2, LOOKUP_TABLE_SIZE  ; Carrega o tamanho da tabela para o laço
    ldi ZL, low(LOOKUP_TBL_ADDR)  ; Z aponta para a tabela de referência
    ldi ZH, high(LOOKUP_TBL_ADDR)
CHECK_LOOKUP_LOOP:
    ld temp, Z                    ; Carrega caractere da tabela de referência
    cp temp, char_atual           ; Compara com o caractere atual
    brne CHECK_LOOKUP_NEXT        ; Se não for igual, pula
    ldi flag_encontrado, 1        ; Se for igual, seta a flag para 1
    rjmp CHECK_LOOKUP_END         ; E termina a busca
CHECK_LOOKUP_NEXT:
    adiw Z, 1                     ; Próximo caractere na tabela de referência
    dec temp2                     ; Decrementa o contador de loop
    brne CHECK_LOOKUP_LOOP        ; Se não for zero, continua
CHECK_LOOKUP_END:

    ;-- ARMAZENAMENTO DO RESULTADO NA TABELA DE SAÍDA --
    ; X já aponta para a posição correta na tabela de saída
    st X+, char_atual             ; 1. Salva o código ASCII do caractere
    st X+, contador               ; 2. Salva a quantidade de ocorrências
    st X+, flag_encontrado        ; 3. Salva a flag (0 ou 1)

    ;-- Passa para o próximo caractere da frase
    adiw Y, 1
    rjmp PROCESS_LOOP

PROCESS_END:
    ret

;---------------------------------------------------------------------------
; Sub-rotina: Copia um bloco de dados da Flash para a RAM
; Entrada: Z -> Endereço de origem na Flash (*2)
;          X -> Endereço de destino na RAM
;          temp (r16) -> Número de bytes a copiar
;---------------------------------------------------------------------------
COPY_FLASH_TO_RAM:
    cpi temp, 0
    breq COPY_DONE
    lpm temp2, Z+   ; Carrega byte da memória de programa e incrementa Z
    st X+, temp2    ; Armazena byte na RAM e incrementa X
    dec temp
    rjmp COPY_FLASH_TO_RAM
COPY_DONE:
    ret
