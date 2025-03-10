# Teste 4: Arquitetura e Web Scraping para Extração de Dados de PDFs

## Cenário
Precisamos automatizar a extração de dados de documentos PDF disponíveis em portais governamentais e sites públicos. O sistema deve baixar os PDFs, converter para texto, extrair informações específicas (nomes próprios e documentos), associar dados ao PDF original e armazenar em banco de dados, tudo isso com baixo custo operacional para processar milhões de documentos.

## Parte 1: Definição da Arquitetura

### Visão Geral da Arquitetura

Proponho uma arquitetura distribuída e orientada a eventos, com componentes específicos para cada fase do processo:

```
┌─────────────┐    ┌───────────────┐    ┌────────────────┐    ┌────────────────┐
│ Orquestrador│    │   Workers de  │    │  Workers de    │    │  Workers de    │
│  de Coleta  │───▶│Web Scraping   │───▶│ Processamento  │───▶│   Extração     │
└─────────────┘    └───────────────┘    └────────────────┘    └────────────────┘
       │                   │                    │                     │
       ▼                   ▼                    ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Barramento de Eventos                          │
└─────────────────────────────────────────────────────────────────────────────┘
       ▲                   ▲                    ▲                     ▲
       │                   │                    │                     │
┌─────────────┐    ┌───────────────┐    ┌────────────────┐    ┌────────────────┐
│             │    │               │    │                │    │                │
│  API REST   │    │ Armazenamento │    │   Banco de    │    │  Monitoramento │
│             │    │               │    │     Dados     │    │                │
└─────────────┘    └───────────────┘    └────────────────┘    └────────────────┘
```

### Servidores e Componentes

1. **Orquestrador de Coleta**:
   - Serviço responsável por gerenciar a agenda de coletas.
   - Distribuição de tarefas para os workers.
   - Controle de prioridade e rebalanceamento de carga.

2. **Workers de Web Scraping**:
   - Instâncias escaláveis que executam o scraping.
   - Utilizam navegadores headless para interação com portais.
   - Autenticação, resolução de captchas e navegação.

3. **Workers de Processamento**:
   - Conversão de PDF para texto.
   - Otimização e normalização do texto extraído.

4. **Workers de Extração**:
   - Identificação de entidades (nomes, documentos).
   - Aplicação de algoritmos de associação contextual.
   - Validação de documentos extraídos (checksums de CPF/CNPJ).

5. **Armazenamento**:
   - Armazenamento de documentos originais e processados.
   - Sistema de cache para arquivos frequentemente acessados.

6. **Banco de Dados**:
   - Armazenamento estruturado dos dados extraídos.
   - Índices otimizados para consultas.

7. **API REST**:
   - Interface para consulta de dados extraídos.
   - Endpoints para monitoramento e controle do sistema.

8. **Monitoramento**:
   - Coleta de métricas de desempenho.
   - Alertas para falhas ou gargalos.

### Escolha de Infraestrutura: Híbrida (Cloud + On-Premises)

**Justificativa para solução híbrida**:
- Componentes intensivos em processamento (OCR, scraping) em servidores dedicados on-premises para controle de custos.
- Componentes que exigem alta disponibilidade e escalabilidade (API, orquestrador) em cloud.
- Armazenamento de objetos em cloud com políticas de lifecycle para redução de custos.

**Estrutura On-Premises**:
- Servidores dedicados para workers de scraping e processamento.
- Configurados com GPUs para aceleração de processamento de OCR quando necessário.
- Conexão dedicada com a cloud para transferência eficiente de dados.

**Estrutura Cloud**:
- Orquestrador e API em instâncias gerenciadas (ex.: AWS ECS, GCP Cloud Run).
- Banco de dados como serviço gerenciado.
- Object storage com camadas de acesso para otimização de custos.

### Estratégia para Escalabilidade e Paralelismo

1. **Arquitetura baseada em filas**:
   - Cada etapa do processamento utiliza filas para distribuição de trabalho.
   - Permite escalar cada componente independentemente conforme demanda.

2. **Processamento por lotes**:
   - Agrupamento de documentos similares para processamento em batch.
   - Otimização de recursos de hardware durante o processamento.

3. **Escalonamento horizontal controlado**:
   - Aumento/redução automática de workers baseado em métricas de carga.
   - Limites máximos definidos para controle de custos.

4. **Priorização de tarefas**:
   - Sistema de priorização para processar documentos mais relevantes primeiro.
   - Distribuição de carga em horários de menor demanda para tarefas não urgentes.

### Mecanismo de Balanceamento e Recuperação

1. **Balanceamento de carga**:
   - Distribuição de tarefas considerando capacidade atual dos workers.
   - Afinidade de sessão para tarefas relacionadas ao mesmo portal.

2. **Circuit Breaker para portais**:
   - Detecção automática de portais indisponíveis.
   - Redirecionamento de recursos para portais funcionais.

3. **Dead Letter Queue**:
   - Isolamento de tarefas problemáticas para análise posterior.
   - Reprocessamento automático com estratégias diferentes.

4. **Checkpointing**:
   - Registro de estado de progresso para permitir retomada após falhas.
   - Recuperação a partir do último ponto válido.

## Parte 2: Tecnologias e Ferramentas

### Linguagens

1. **Node.js**:
   - Justificativa: Arquitetura orientada a eventos ideal para operações I/O intensivas.
   - Ecossistema rico para web scraping (Puppeteer, Playwright, Cheerio).
   - Eficiente para processamento assíncrono e programação reativa.
   - Ampla comunidade e bibliotecas para todos os aspectos do projeto.

2. **TypeScript**:
   - Justificativa: Tipagem estática para maior robustez e manutenibilidade.
   - Melhor suporte a projetos de grande escala.
   - Facilita refatoração e documentação do código.

### Bibliotecas e Frameworks

1. **Scraping**:
   - Puppeteer/Playwright: automação avançada de navegador com suporte Chrome e Firefox.
   - Cheerio: parsing e manipulação de HTML para scraping leve.
   - 2captcha/Anti-Captcha API: para resolução automática de captchas.
   - Axios/Got: para requisições HTTP eficientes.

2. **Processamento de PDF**:
   - pdf-parse/pdf.js: extração eficiente de texto de PDFs.
   - node-tesseract-ocr: integração com Tesseract para OCR quando necessário.
   - pdf-img-convert: conversão de PDF para imagens para OCR.

3. **Extração de Entidades**:
   - natural: biblioteca de NLP em JavaScript para processamento de texto.
   - compromise: framework leve de NLP para reconhecimento de entidades.
   - RegEx otimizados: para extração de padrões de documentos.
   - Biblioteca customizada: para validação de documentos brasileiros (CPF/CNPJ).

4. **Mensageria**:
   - Apache Kafka (com kafkajs): para comunicação entre componentes.
   - Bull/BullMQ: filas baseadas em Redis para processamento de tarefas.
   - Redis Streams: alternativa leve para filas menores.

### Banco de Dados

**Principal: PostgreSQL + TimescaleDB**
- Justificativa:
  - Suporte a dados JSON para armazenamento de textos extraídos.
  - Extensão TimescaleDB para eficiência com dados temporais (logs de processamento).
  - Particionamento nativo para tabelas grandes.
  - Capacidade de indexação de texto completo.

**Complementar: Elasticsearch**
- Justificativa:
  - Buscas complexas em texto extraído.
  - Indexação eficiente para consultas frequentes.
  - Capacidade de busca fuzzy para nomes similares.

### Armazenamento

1. **Documentos Originais**:
   - MinIO (on-premises): armazenamento de objetos compatível com S3.
   - AWS S3 com Intelligent Tiering (cloud): para otimização automática de custos.

2. **Dados Processados**:
   - PostgreSQL para metadados e relações.
   - Object storage para textos extraídos completos.

### Cache

1. **Redis**:
   - Cache de resultados de processamento.
   - Armazenamento de sessões de scraping.
   - Rate limiting para acessos a portais.

2. **Sistema de cache distribuído em memória**:
   - Hazelcast/Memcached para cache compartilhado entre workers.

### Monitoramento

1. **Prometheus + Grafana**:
   - Coleta de métricas de performance.
   - Dashboards para visualização de gargalos.

2. **ELK Stack (Elasticsearch, Logstash, Kibana)**:
   - Análise centralizada de logs.
   - Detecção de padrões de falha.

3. **Jaeger/Zipkin**:
   - Tracing distribuído para identificação de gargalos.

## Parte 3: Modelagem do Banco de Dados

### Modelo Conceitual

```
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│    Documento  │      │    Entidade   │      │  TipoEntidade │
│───────────────│      │───────────────│      │───────────────│
│ id            │      │ id            │      │ id            │
│ hash          │◄─────┼─documento_id  │      │ nome          │
│ uri_origem    │      │ texto         │      │ regex_validacao
│ data_coleta   │      │ tipo_id       │◄─────┼───────────────┘
│ caminho_s3    │      │ confiabilidade│      │
│ texto_extraido│      │ pagina        │      │
│ status        │      │ posicao_x     │      │
│ metadados     │      │ posicao_y     │      │
└───────────────┘      └───────────────┘      │
        │                      ▲               │
        │                      │               │
        ▼                      │               │
┌───────────────┐      ┌───────────────┐      │
│   Contexto    │      │   Relacao     │      │
│───────────────│      │───────────────│      │
│ id            │      │ id            │      │
│ documento_id  │      │ entidade_id_1 │      │
│ texto         │      │ entidade_id_2 │      │
│ pagina        │      │ tipo_relacao  │      │
│ indice_inicio │      │ confiabilidade│      │
│ indice_fim    │      │ contexto_id   │◄─────┘
└───────────────┘      └───────────────┘
```

### Detalhamento das Tabelas

1. **Documento**:
   - Armazena metadados do PDF original.
   - Campo `texto_extraido` pode ser externalizado para storage de objetos com referência.
   - Índices em `hash` (para detecção de duplicatas) e `status` (para consultas de processamento).
   - Particionamento por data_coleta para consultas eficientes.

2. **Entidade**:
   - Armazena nomes e documentos extraídos.
   - `tipo_id` identifica se é nome, CPF, CNPJ, etc.
   - `confiabilidade` indica o nível de certeza da extração (0-100%).
   - Índices em `documento_id` e `tipo_id` para consultas frequentes.

3. **TipoEntidade**:
   - Catálogo de tipos possíveis (Nome, CPF, CNPJ, RG, etc.).
   - `regex_validacao` auxilia na validação durante extração.

4. **Contexto**:
   - Armazena o contexto textual onde a entidade foi encontrada.
   - Essencial para análise de relações entre entidades.
   - Referencia posição exata no texto para auditoria.

5. **Relacao**:
   - Estabelece conexões entre entidades (ex: Nome ↔ CPF).
   - `tipo_relacao` define natureza da relação.
   - `confiabilidade` indica probabilidade da relação ser correta.
   - Referência ao contexto que suporta esta relação.

### Otimização para Consultas

1. **Índices Compostos**:
   - `(documento_id, tipo_id)` para rápida recuperação de entidades por tipo.
   - `(entidade_id_1, entidade_id_2)` para consultas de relações bidirecionais.

2. **Particionamento**:
   - Tabela `Documento` particionada por `data_coleta` (mensal).
   - Tabela `Entidade` particionada por `tipo_id` para balancear volume.

3. **Índices de Texto Completo**:
   - Índice GIN para `texto_extraido` em `Documento`.
   - Índice GIN para `texto` em `Contexto`.

4. **Materialização de Visões**:
   - Visões materializadas para consultas frequentes.
   - Atualização programada em horários de baixo uso.

### Integridade e Consistência

1. **Constraints**:
   - Foreign keys entre todas as tabelas relacionadas.
   - Validações em nível de aplicação para documentos (ex: algoritmo de validação de CPF/CNPJ).

2. **Transações**:
   - Uso de transações para garantir atomicidade nas inserções relacionadas.
   - Isolamento adequado para evitar inconsistências durante leituras/escritas concorrentes.

3. **Versionamento**:
   - Campo de versão em `Documento` para controle de atualizações.
   - Histórico de alterações em tabela separada para auditoria.

## Parte 4: Método para Associação de Nomes e Documentos

### Abordagem Multi-nível para Associação

1. **Análise de Proximidade Espacial**:
   - Documentos próximos a nomes no layout do PDF têm maior probabilidade de associação.
   - Implementação de algoritmo de janela deslizante para identificar padrões de layout.

2. **Análise de Contexto Linguístico**:
   - Identificação de padrões textuais que indicam relação.
   - Exemplos: "CPF: XXX.XXX.XXX-XX do Sr. João Silva", "portador do RG nº XXX".
   - Uso de técnicas de NLP para reconhecimento de padrões.

3. **Modelo de Machine Learning**:
   - Modelo treinado para classificar probabilidade de relação entre entidades.
   - Features: distância no texto, presença de palavras-chave de relação, posicionamento relativo.

4. **Sistema de Pontuação (Scoring)**:
   - Combinação ponderada dos métodos anteriores.
   - Threshold mínimo de confiança para estabelecer relação.
   - Diferentes pesos para diferentes fontes/tipos de documento.

### Algoritmo de Associação

```javascript
// Pseudocódigo do algoritmo de associação em JavaScript
async function associarEntidades(documento) {
  // Identificar todas as entidades
  const nomes = await extrairEntidadesTipo(documento, "NOME");
  const documentos = await extrairEntidadesTipo(documento, ["CPF", "CNPJ", "RG"]);
  
  const relacoes = [];
  
  // Para cada nome, buscar documentos potencialmente relacionados
  for (const nome of nomes) {
    // 1. Analisar proximidade espacial
    const candidatosEspacial = await encontrarProximos(nome, documentos, { maxDistancia: 500 });
    
    // 2. Analisar contexto linguístico
    const candidatosContexto = await encontrarPorContexto(nome, documentos);
    
    // 3. Aplicar modelo ML para refinar associações
    const candidatosUnicos = [...new Set([...candidatosEspacial, ...candidatosContexto])];
    
    for (const doc of candidatosUnicos) {
      const score = await calcularScoreAssociacao(nome, doc, documento);
      
      if (score > THRESHOLD_MINIMO) {
        relacoes.push({
          nome_id: nome.id,
          documento_id: doc.id,
          score: score,
          contexto: await extrairContexto(nome, doc, documento)
        });
      }
    }
  }
  
  return relacoes;
}
```

### Validação de Documentos

1. **Validação Sintática**:
   - Aplicação de algoritmos de validação específicos (ex: dígitos verificadores de CPF/CNPJ).
   - Eliminação de falsos positivos (números que parecem documentos mas não são).

2. **Validação Contextual**:
   - Verificação se o contexto suporta que o número é realmente um documento.
   - Presença de palavras-chave ou formatação típica.

3. **Cross-Referência**:
   - Comparação com dados já conhecidos de associações prévias.
   - Construção gradual de base de conhecimento.

## Parte 5: Estratégia para Baixo Custo

### Minimização de Custos com Servidores

1. **Arquitetura Híbrida**:
   - Componentes estáveis e previsíveis em servidores dedicados.
   - Componentes com demanda variável em cloud com auto-scaling controlado.

2. **Servidores Spot/Preemptive**:
   - Uso de instâncias spot da AWS ou preemptive do GCP para workers não críticos.
   - Economia de 60-90% nos custos de computação.

3. **Reserva de Capacidade**:
   - Compra de instâncias reservadas para cargas de base.
   - Complemento com instâncias sob demanda apenas para picos.

4. **Auto-Scaling Inteligente**:
   - Algoritmos preditivos para acionar escalabilidade antes de picos esperados.
   - Desligamento proativo de recursos ociosos.

### Evitar Desperdício de Processamento

1. **Detecção de Duplicatas**:
   - Cálculo de hash de documentos antes do processamento completo.
   - Skip de processamento para documentos já conhecidos.

2. **Processamento em Camadas**:
   - Extração básica de texto para todos os documentos.
   - OCR apenas quando necessário (texto não extraível diretamente).
   - Processamento avançado apenas para documentos relevantes.

3. **Priorização Inteligente**:
   - ML para prever probabilidade de documento conter informações relevantes.
   - Processamento prioritário dos mais promissores.

4. **Batching Eficiente**:
   - Agrupamento de tarefas similares para otimizar uso de recursos.
   - Processamento em lotes durante horários de menor custo (cloud) ou menor carga (on-premises).

### Otimização de Armazenamento

1. **Políticas de Lifecycle**:
   - Migração automática para classes de armazenamento mais baratas para dados menos acessados.
   - Exemplo: S3 Standard → S3 Infrequent Access → S3 Glacier.

2. **Compressão Seletiva**:
   - Compressão de PDFs e textos extraídos raramente acessados.
   - Algoritmos de compressão otimizados para textos.

3. **Deduplicação de Dados**:
   - Armazenamento único de documentos idênticos com referências.
   - Deduplicação em nível de blocos para otimização adicional.

4. **Retenção Baseada em Valor**:
   - Política de retenção variável baseada na relevância dos documentos.
   - Exclusão ou arquivamento profundo de dados com baixo valor após período definido.

### Soluções Open-Source

1. **Stack Tecnológico**:
   - PostgreSQL em vez de soluções proprietárias de banco de dados.
   - Apache Kafka em vez de serviços gerenciados de mensageria.
   - Elasticsearch em vez de serviços de busca proprietários.

2. **Containerização com Kubernetes**:
   - Orquestração eficiente de recursos.
   - Portabilidade entre ambientes on-premises e cloud.

3. **Monitoramento e Observabilidade**:
   - Prometheus/Grafana em vez de soluções proprietárias de APM.
   - ELK Stack para gestão centralizada de logs.

4. **Implementações Customizadas**:
   - Desenvolvimento de componentes específicos otimizados para o caso de uso.
   - Mais eficientes que soluções genéricas em termos de recursos.

## Considerações Finais

A arquitetura proposta equilibra o requisito de baixo custo com a necessidade de processar milhões de documentos, utilizando técnicas de otimização em cada nível do sistema. A abordagem híbrida (cloud + on-premises) oferece o melhor dos dois mundos, permitindo controle de custos sem sacrificar escalabilidade. As estratégias de associação de nomes e documentos combinam métodos determinísticos e probabilísticos para alcançar alta precisão, mesmo em contextos desafiadores.

A implementação gradual, começando pelos componentes centrais e expandindo para funcionalidades mais avançadas, permitirá validar o sistema e ajustar a arquitetura conforme necessário, minimizando riscos e otimizando o ROI do projeto.
