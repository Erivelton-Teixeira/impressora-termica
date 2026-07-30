# Java - configurando uma impressora térmica
Tutorial para configração de uma impressora térmica de (cupom fiscal) usando a linguagem Java.
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/11a2b61c-4a4c-42f4-a0e3-4585c0d217e0" />

# Impressão em Impressora Térmica via Socket (ESC/POS) — Java

> ---
> ### 📌 Selo de Metadados
> | Campo | Valor |
> |---|---|
> | **Título** | Impressão em Impressora Térmica via Socket (ESC/POS) — Java |
> | **Autor** | Erivelton Teixeira |
> | **Instituição** | Senac Tatuapé |
> | **Linguagem** | Java (JDK 8+) |
> | **Protocolo** | ESC/POS sobre TCP/IP (porta 9100 — raw printing) |
> | **Equipamento testado** | Impressora térmica de cupom, interface Ethernet (WebConfig v1.02) |
> | **IP do dispositivo** | 10.26.49.38 (Fixed IP) |
> | **MAC Address** | 00-31-81-2A-94-FD |
> | **Data de geração do documento** | 30/07/2026 |
> | **Versão do documento** | 1.1 |
> | **Status** | ✅ Testado e validado (impressão física confirmada) |
> ---

## 1. Objetivo

Este documento descreve o funcionamento da classe `Impressora`, um programa Java que se conecta a uma impressora térmica de cupom (rede local) via **Socket TCP/IP** e envia comandos no padrão **ESC/POS** para formatar e imprimir um cupom de teste.

O código serve como material de estudo sobre:
- Comunicação via `Socket` em Java;
- Envio de dados binários (`OutputStream`) para dispositivos de rede;
- Uso do protocolo **ESC/POS**, padrão de comandos para impressoras térmicas/fiscais;
- Manipulação de datas com a API `java.time`;
- Codificação de caracteres (`CP850`) para impressoras térmicas.

---

## 2. Pacote e Imports

```java
package impressora;

import java.net.Socket;
import java.io.OutputStream;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
```

| Import | Finalidade |
|---|---|
| `java.net.Socket` | Cria uma conexão TCP/IP com um endereço IP e porta específicos — neste caso, a impressora de rede. |
| `java.io.OutputStream` | Fluxo de saída de bytes, usado para **enviar** os dados/comandos à impressora. |
| `java.time.LocalDateTime` | Representa data e hora atuais do sistema, sem fuso horário. |
| `java.time.format.DateTimeFormatter` | Formata o objeto `LocalDateTime` em uma `String` legível (ex.: `dd/MM/yyyy HH:mm:ss`). |

---

## 3. Conceito-chave: ESC/POS

**ESC/POS** é uma linguagem de comandos (criada pela Epson e adotada como padrão de mercado) usada para controlar impressoras térmicas de cupom fiscal/não fiscal. Os comandos são enviados como sequências de **bytes** (geralmente iniciados pelo caractere `ESC` = `0x1B` ou `GS` = `0x1D`), interpretados diretamente pelo firmware da impressora.

Diferente de uma impressora comum (que recebe um documento renderizado), a impressora térmica recebe **texto puro + comandos binários intercalados**.

---

## 4. Conexão com a Impressora

```java
Socket impressora = new Socket("10.26.49.38", 9100);
OutputStream saida = impressora.getOutputStream();
```

- A impressora é acessada como se fosse um servidor de rede: **IP + porta**.
- A **porta 9100** é o padrão *de facto* para impressão bruta (*raw printing*) via TCP/IP (usada por praticamente todas as impressoras térmicas e muitas laser/matriciais compatíveis com rede).
- `getOutputStream()` retorna o canal por onde os bytes serão enviados à impressora.

> **Boa prática de estudo:** o IP `10.26.49.38` é fixo no código (hardcoded). Em um cenário real, seria interessante externalizar essa configuração (arquivo `.properties`, variável de ambiente, etc.).

### 4.1 Configuração de rede via WebConfig

Antes de utilizar o Socket em Java, o IP da impressora precisa ser configurado no próprio equipamento. Isso é feito pela interface web embarcada (**Ethernet WebConfig**), acessível pelo navegador digitando o IP atual do dispositivo.

**Tela de Configuração** — definição de IP fixo, máscara de sub-rede e gateway:
<img width="802" height="708" alt="image" src="https://github.com/user-attachments/assets/57c68384-5015-4156-902c-080661f269f3" />


Nesta tela foi definido o **Fixed IP Address**: `10.26.49.38`, máscara `255.255.255.0` e gateway `10.26.49.1` — os mesmos valores usados no construtor `new Socket("10.26.49.38", 9100)`.

**Tela de Informação** — confirmação dos dados de rede aplicados:

<img width="819" height="717" alt="image" src="https://github.com/user-attachments/assets/162bb22c-d7be-421a-8ed7-947b8f0bcded" />

Essa tela exibe o resumo do estado atual da interface de rede da impressora, incluindo o **MAC Address** (`00-31-81-2A-94-FD`), o **IP Address**, a **Subnet Mask**, o **Gateway** e o status do DHCP — útil para conferir se a configuração foi aplicada corretamente antes de testar a conexão via código.

---

## 5. Inicialização da Impressora

```java
saida.write(new byte[] {0x1B, 0x40});
```

| Bytes | Comando ESC/POS | Efeito |
|---|---|---|
| `0x1B 0x40` | `ESC @` | **Inicializa/reseta** a impressora, limpando buffer e restaurando configurações padrão (fonte, alinhamento, espaçamento). |

Essa é sempre a primeira instrução enviada, garantindo que o cupom comece de um estado conhecido.

---

## 6. Impressão de Data e Hora

```java
LocalDateTime agora = LocalDateTime.now();
DateTimeFormatter formato = DateTimeFormatter.ofPattern("dd/MM/yyyy  HH:mm:ss");
String datahora = "Data: " + agora.format(formato) + "\n\n";
saida.write(datahora.getBytes("CP850"));
```

- `LocalDateTime.now()` captura o instante atual do relógio do sistema.
- `DateTimeFormatter` converte esse objeto para o formato `dd/MM/yyyy HH:mm:ss` (ex.: `30/07/2026 14:32:10`).
- `getBytes("CP850")` converte a `String` para um array de bytes usando a codificação **CP850** (Code Page 850 — "Latin 1"), padrão suportado pela maioria das impressoras térmicas para caracteres acentuados (á, ã, ç, etc.). Usar `UTF-8` aqui normalmente resultaria em caracteres corrompidos na impressão.
- `\n\n` insere uma linha em branco após a data.

---

## 7. Formatação de Texto — Comandos ESC/POS Utilizados

### 7.1 Alinhamento

```java
saida.write(new byte[] {0x1B, 0x61, 0x01}); // centralizado
saida.write(new byte[] {0x1B, 0x61, 0x00}); // esquerda
// saida.write(new byte[] {0x1B, 0x61, 0x02}); // direita (comentado)
```

| Comando | Byte final | Alinhamento |
|---|---|---|
| `ESC a` | `0x00` | Esquerda (padrão) |
| `ESC a` | `0x01` | Centro |
| `ESC a` | `0x02` | Direita |

### 7.2 Tamanho da fonte

```java
saida.write(new byte[] {0x1D, 0x21, 0x11}); // fonte ampliada
// ... texto ...
saida.write(new byte[] {0x1D, 0x21, 0x00}); // fonte normal
```

| Comando | Byte final | Efeito |
|---|---|---|
| `GS !` | `0x00` | Tamanho normal |
| `GS !` | `0x11` | Largura x2 e altura x2 |
| `GS !` | `0x22` | Largura x3 e altura x3 |

> O byte final combina os *nibbles* de altura e largura (ex.: `0x11` = altura 1 + largura 1, ou seja, "2x" em ambas dimensões).

### 7.3 Negrito

```java
saida.write(new byte[] {0x1B, 0x45, 0x01}); // ativa negrito
// ... texto ...
saida.write(new byte[] {0x1B, 0x45, 0x00}); // desativa negrito
```

| Comando | Byte final | Efeito |
|---|---|---|
| `ESC E` | `0x01` | Ativa negrito |
| `ESC E` | `0x00` | Desativa negrito |

---

## 8. Conteúdo Impresso

Sequência de textos enviados ao cupom, já formatados conforme os comandos acima:

1. **Data/hora** (alinhamento padrão, fonte normal);
2. **"HELLO WORLD!"** — centralizado, fonte grande (título);
3. **"Senac Tatuapé"** — alinhado à esquerda, negrito;
4. **"Erivelton Teixeira"** — alinhado à esquerda, fonte normal (sem negrito).

Cada bloco de texto é convertido para bytes com `getBytes("CP850")` antes do envio.

---

## 8.1 Resultado do teste de impressão

Evidência física do cupom gerado a partir da execução do programa (conteúdo ilustrativo de teste, com logotipo em vez de texto puro):

<img width="1024" height="768" alt="image" src="https://github.com/user-attachments/assets/cf831adb-6838-4faa-9d7b-9097bfd433fa" />


O teste confirma o funcionamento do fluxo completo: **conexão via Socket → envio de comandos ESC/POS → renderização e corte físico do papel** pela impressora térmica.

---

## 9. Avanço de Papel e Corte

```java
saida.write(new byte[] {0x1B, 0x64, 0x05}); // avança 5 linhas
saida.write(new byte[] {0x1D, 0x56, 0x00}); // corte total do papel
```

| Comando | Descrição |
|---|---|
| `ESC d n` | Avança o papel em `n` linhas (aqui, `0x05` = 5 linhas), garantindo espaço para o corte sem cortar o texto. |
| `GS V 0` | Executa o **corte total** do papel (guilhotina). Existe também o corte parcial (`0x01`), que deixa uma pequena parte unida. |

---

## 10. Envio e Encerramento

```java
saida.flush();
impressora.close();
```

- `flush()` força o envio imediato de todos os bytes armazenados em buffer para a impressora (sem isso, os dados poderiam ficar retidos e nunca ser efetivamente transmitidos).
- `close()` encerra a conexão Socket, liberando o recurso de rede.

---

## 11. Tratamento de Exceções

```java
try {
    // ... lógica de conexão e impressão ...
} catch (Exception e) {
    System.out.println(e);
}
```

Todo o processo está envolvido em um bloco `try/catch` genérico, pois diversas operações podem lançar exceções verificadas (*checked exceptions*), como:

- `UnknownHostException` / `IOException` — falha ao conectar (IP incorreto, impressora desligada, rede indisponível);
- `UnsupportedEncodingException` — codificação `CP850` não suportada pela JVM (raro, mas possível).

> **Observação de estudo:** capturar `Exception` genérica funciona para fins didáticos, mas em código de produção é recomendável tratar exceções específicas separadamente, além de registrar logs mais detalhados (`e.printStackTrace()` ou um logger apropriado).

---

## 12. Tabela-resumo dos comandos ESC/POS usados

| Bytes | Comando | Função |
|---|---|---|
| `0x1B 0x40` | ESC @ | Inicializa a impressora |
| `0x1B 0x61 n` | ESC a n | Alinhamento (0=esq, 1=centro, 2=dir) |
| `0x1D 0x21 n` | GS ! n | Tamanho da fonte |
| `0x1B 0x45 n` | ESC E n | Negrito (0=off, 1=on) |
| `0x1B 0x64 n` | ESC d n | Avança `n` linhas |
| `0x1D 0x56 0x00` | GS V 0 | Corte total do papel |

---

## 13. Código-fonte completo (referência)

```java
package impressora;

import java.net.Socket;
import java.io.OutputStream;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class Impressora {

    public static void main(String[] args) {
        try {
            Socket impressora = new Socket("10.26.49.38", 9100);
            OutputStream saida = impressora.getOutputStream();

            // Inicializa a impressora (reset)
            saida.write(new byte[] {0x1B, 0x40});

            // Data e hora
            LocalDateTime agora = LocalDateTime.now();
            DateTimeFormatter formato = DateTimeFormatter.ofPattern("dd/MM/yyyy  HH:mm:ss");
            String datahora = "Data: " + agora.format(formato) + "\n\n";
            saida.write(datahora.getBytes("CP850"));

            // Centraliza
            saida.write(new byte[] {0x1B, 0x61, 0x01});

            // Fonte grande
            saida.write(new byte[] {0x1D, 0x21, 0x11});
            saida.write("HELLO WORLD!\n\n".getBytes("CP850"));
            saida.write(new byte[] {0x1D, 0x21, 0x00});

            // Alinha à esquerda
            saida.write(new byte[] {0x1B, 0x61, 0x00});

            // Negrito
            saida.write(new byte[] {0x1B, 0x45, 0x01});
            saida.write("Senac Tatuapé\n".getBytes("CP850"));
            saida.write(new byte[] {0x1B, 0x45, 0x00});
            saida.write("Erivelton Teixeira\n\n".getBytes("CP850"));

            // Avança papel
            saida.write(new byte[] {0x1B, 0x64, 0x05});

            // Corte
            saida.write(new byte[] {0x1D, 0x56, 0x00});

            saida.flush();
            impressora.close();

        } catch (Exception e) {
            System.out.println(e);
        }
    }
}
```

---

## 14. Possíveis melhorias (para estudo futuro)

- Externalizar IP e porta da impressora em arquivo de configuração;
- Criar métodos reutilizáveis (ex.: `centralizar()`, `negrito(boolean)`, `imprimirTexto(String)`) em vez de repetir arrays de bytes;
- Tratar exceções específicas (`IOException`, `UnknownHostException`) separadamente;
- Adicionar timeout de conexão (`Socket` possui construtor com `connect(SocketAddress, int timeout)`);
- Validar se a impressora está disponível antes de tentar imprimir.

---

## 15. Referências

- Especificação de comandos ESC/POS (Epson) — referência oficial de fabricantes de impressoras térmicas.
- Documentação oficial Java: `java.net.Socket`, `java.io.OutputStream`, `java.time.LocalDateTime`.

---
