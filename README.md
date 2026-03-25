# Dashboard de Desempenho WFM (Workforce Management) 📊

Este repositório documenta a arquitetura de um projeto de **Workforce Management (WFM)** construído nativamente no Google Sheets. O objetivo da ferramenta é automatizar o cruzamento de logs de sistemas de telefonia (Twilio) e atendimento (Zendesk/CSAT) para gerar visões executivas de produtividade e SLA.

*(Imagens do dashboard e da base de dados serão adicionadas em breve na pasta `/assets`)*

## 🚀 O Problema de Negócio

Empresas de atendimento e suporte precisam medir a eficiência de seus analistas. Para isso, é necessário saber:
1. **Horas Trabalhadas:** O agente cumpriu a escala de 08:20:00?
2. **Tempo Improdutivo:** Quanto tempo o agente passou em pausas sistêmicas, problemas técnicos ou desconectado?
3. **Volume de Atendimentos:** Quantas tratativas foram finalizadas no dia?

Os dados vêm de fontes distintas e com formatos de data/hora diferentes. Este dashboard atua como uma ferramenta de **ETL (Extract, Transform, Load)** usando apenas fórmulas do Sheets.

## 🏗️ Arquitetura das Planilhas

A estrutura foi dividida para garantir a performance do Sheets (evitando o travamento por excesso de fórmulas voláteis):
- **Base Automatizada (Twilio):** Recebe os logs brutos de mudança de status (`Disponível`, `Pré-Almoço`, `Sistema`, `Task`, etc).
- **Base de Atendidas:** Recebe o consolidado numérico de contatos resolvidos.
- **Resultado de Desempenho:** A camada de visualização (Front-end), onde as fórmulas de agregação são processadas.

## 🧠 Dicionário de Fórmulas e Lógicas Aplicadas

Para evitar o uso de Scripts externos, construí lógicas matemáticas complexas utilizando as funções modernas do Google Sheets:

### 1. Classificação de Pausas Improdutivas
Uso de lógica condicional alinhada para rotular tempos com base na regra de negócio da empresa.
```excel
=SE(OU(F2="Particular"; F2="Banco de Horas"; F2="Tok- Pausa 15"); "Contar"; 
  SE(F2="Desconectado"; 
    SE(E(D2=D1; OU(F1="Almoço"; F1="Pré-Almoço")); "Ignorar"; "Contar"); 
  "Ignorar"))
```
*Lógica: Filtra pausas oficiais e lida com o bug do analista ficar "Desconectado" logo após voltar do "Almoço".*

### 2. Conversão de Milissegundos (Unix) para Formato de Hora (HH:MM:SS)
```excel
=G2/86400
```
*Lógica: Divide a quantidade de segundos reportada pela Twilio pelo total de segundos de um dia (86400) para o Google Sheets formatar perfeitamente como Horas.*

### 3. Extração Dinâmica (Log In e Log Out)
```excel
=SEERRO(LET(
    ultima_linha; ÍNDICE(SORT(FILTER('base automatizada'!A:J; ARRUMAR('base automatizada'!D:D)=ARRUMAR(B7)); 3; FALSO); 1; 0);
    atividade; ÍNDICE(ultima_linha; 6);
    hora_inicio; ÍNDICE(ultima_linha; 3);
    duracao; ÍNDICE(ultima_linha; 10);

    SE(atividade="Desconectado"; hora_inicio; hora_inicio + duracao)
))
```
*Lógica: A função `LET` aloca variáveis na memória (otimizando processamento). Usa `FILTER` e `SORT` para encontrar o último registro exato de um atendente no dia, definindo com precisão sua hora de saída oficial.*

### 4. Cálculo de Banco de Horas (SLA de 08:20:00)
```excel
=SE(H7="Sem Débito"; G7-"08:20:00"; "Sem BH")
```
*Lógica: Deduz o tempo trabalhado do contrato base (8 horas e 20 minutos). Se exceder, contabiliza o Banco de Horas Realizado.*

### 5. Agrupamento de Atividades (SOMASES Múltiplos)
```excel
=SOMASES('base automatizada'!J:J; 'base automatizada'!D:D; B7; 'base automatizada'!F:F; "Treinamento") + 
 SOMASES('base automatizada'!J:J; 'base automatizada'!D:D; B7; 'base automatizada'!F:F; "Reunião") + ...
```
*Lógica: Soma os tempos condicionalmente usando como chaves primárias o Nome do Agente e a Categoria de Pausa (Atividades Operacionais).*

---
**Nota Técnica:** Este modelo gerencial demonstrou como reduzir o overhead manual dos coordenadores de operação, criando painéis executivos autossuficientes sem necessidade de custos com ferramentas pagas de BI (como Tableau ou PowerBI).
