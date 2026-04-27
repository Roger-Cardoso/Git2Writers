# **Editor de Texto Semântico**

**Proposta**

Eu quero muito um editor de texto para eu escrever minhas obras literárias:

\- Programado em C++20, GTK4, SQLite3, Meson, Threading: STL, Spellcheck: Hunspell.

\- A Entrada de Texto é armazenada no GTKTextBuffer do GTK4 e Renderiza na TextView, executando em paralelo uma classificação do conteúdo do Buffer para o modelo de entidades e ocorrências.

\- Um Banco para guardar os Escritos normalizados, salvos apenas uma vez, e a cada repetição é feita uma referência ao original, somando as ocorrências.

\- Cada escrito dentro do Banco tem um id que é um hash preparado usando a versão normalizada.

\- Tipos de Entidades:

Palavras → (guarda no banco e marca as ocorrências.)

Espaço → (não guarda no banco, apenas marca as ocorrências.)

Números → (guarda no banco e marca as ocorrências.) Engloba todos os números, Ex: 

Dígitos ‘0’ ‘1’ ‘2’ ‘3’ ‘4’ ‘5’ ‘6’ ‘7’ ‘8’ ‘9’

Valores Positivos ‘123’

Valores Negativos ‘-123’

Números Separados por Classes ‘1.000’ ‘1 000’ ‘1,000’

Números Decimais ‘1,5’ ‘1.5’

Frações  ‘1/2' ‘½’ 

Pontuação → (não guarda no banco, apenas identifica qual é a pontuação e marca as ocorrências.)

Ex: ‘.’ ‘,’ ‘\!’ ‘?’ ‘:’ ‘;’

Operadores de Semântica Computacional → (não guarda no banco, apenas identifica qual é o operador e marca as ocorrências.)

Ex: ‘+’ ‘-’ ‘\*’ ‘/’ ‘=’

Delimitadores Estruturais → (não guarda no banco, apenas identifica qual é o delimitador e marca as ocorrências.)  
Ex: ‘(‘ ’)’ ‘\[’ ‘\]’ ‘{’ ‘}’ ‘\\’ ‘/’ ‘|’ ‘’’ ‘“‘  
Símbolos Residuais → (não guarda no banco, apenas identifica qual é o Símbolo e marca as ocorrências.)   
‘%’ ‘$’ ‘@’ ‘\#’ ‘&’ ‘§’  
Emojis → (não guarda no banco, apenas identifica qual é o Emoji e marca as ocorrências.)   
Comandos Inline → (não guarda no banco, consulta na lista de comandos, apenas identifica qual é o comando, ativa, executa a sua função, desaparece.)   
Fórmulas → (não guarda no banco, consulta na lista de fórmulas, apenas identifica qual é o comando (usando as variáveis fornecidas), ativa, substitui o escrito pelo resultado do comando da fórmula, como no excel). 

\- Localização do texto é uma variável captada pela ocorrência e não uma consequência.

\- Processamento de texto.

\- Formatação.

\- Corretor ortográfico.

\- Trabalha com visualização como primeira prioridade e processamento depois.

\- Trabalha nativamente com projetos e arquivos em Workspaces direto na interface.

\- Na janela do programa, trabalha com um container sem páginas para escrita em arquivos de texto soltos que são considerados entidades e funcionam como blocos de montar.

O conteúdo dos arquivos é mostrado no container, esse arquivo é uma entidade com nome, Id e MetaDados.

\- Possuirá uma funcionalidade de renderização onde eu possa selecionar arquivos e ordená-los para formar estruturas visuais unificadas.

\- Possibilita compilar e exportar essas renderizações.

\- Possibilita criar MetaDados ( Texto linkado ao arquivo narrativo que contém todo o conteúdo informativo de fundo, como: roteiro, direção, estrutura, objetivos e detalhes desse arquivo, documentando e preservando a vontade do autor).

\- Possibilita criar dinamicamente layers globais personalizadas para separar trechos do texto por utilidade, podendo aplicar vários layers ao mesmo texto ( Exemplos de tipos de Layers: Pensamentos, Referências, Background, Avisos, To Do Lists).

\- Possibilita escolher quais layers estarão visíveis ( por padrão, enquanto o texto pertencer a pelo menos 1 layer ativo, ele não deve ser desabilitado, mas deve existir uma opção de forçar a ocultação do texto do layer selecionado para desativação).

**Especificação**

## **0\. Modelo Fundamental**

Separação de três camadas:

* **Texto bruto (fluxo)** → GtkTextBuffer  
* **Modelo semântico** → Entidades \+ Ocorrências  
* **Modelo narrativo** → Estruturas \+ Metadados

Princípio:

* O texto **não contém estrutura narrativa explicitamente**  
* A narrativa é uma **camada paralela e associativa**

---

## **1\. Arquitetura Geral**

### **1.1 Core**

Responsável por três subsistemas independentes:

1. **Engine de Texto**  
2. **Engine Semântica**  
3. **Engine Narrativa**

### **1.2 Pipeline Assíncrono**

Afeta apenas:

* Texto bruto  
* Modelo semântico

Não afeta diretamente:

* Modelo narrativo (manual, orientado ao autor)

Separação obrigatória entre UI e processamento:

* Thread principal:  
  * GTK (UI)  
* Worker threads:  
  * Tokenização  
  * Normalização  
  * Persistência

Comunicação via:

* Filas lock-free (preferencial)  
* Ou filas protegidas por mutex com baixa contenção

Sistema baseado em eventos:

* Alteração no buffer → geração de “dirty range”  
* Enfileiramento para processamento

### **1.3 Interface (GTK4)**

A UI projeta três dimensões simultâneas:

* Texto (edição)  
* Semântica (layers)  
* Narrativa (estruturas)

Regra central:

A UI nunca bloqueia esperando processamento.

---

## **2\. Modelo de Dados**

### **2.1 Entidade (imutável e normalizada)**

* Unidade semântica normalizada  
* Independente de narrativa

struct Entity {  
   uint64\_t id;       
   std::string content;  // forma normalizada  
   EntityType type;  
};

#### **Regra adicional (crítica):**

Em caso de colisão de hash:

* Comparar `content`  
* Apenas considerar igual se o conteúdo for idêntico

---

### **2.2 Tipos de Entidade**

Palavras → (guarda no banco e marca as ocorrências.)

Espaço → (não guarda no banco, apenas marca as ocorrências.)

Números → (guarda no banco e marca as ocorrências.) Engloba todos os números, Ex: 

Dígitos ‘0’ ‘1’ ‘2’ ‘3’ ‘4’ ‘5’ ‘6’ ‘7’ ‘8’ ‘9’

Valores Positivos ‘123’

Valores Negativos ‘-123’

Números Separados por Classes ‘1.000’ ‘1 000’ ‘1,000’

Números Decimais ‘1,5’ ‘1.5’

Frações  ‘1/2' ‘½’ 

Pontuação → (não guarda no banco, apenas identifica qual é a pontuação e marca as ocorrências.)

Ex: ‘.’ ‘,’ ‘\!’ ‘?’ ‘:’ ‘;’

Operadores de Semântica Computacional → (não guarda no banco, apenas identifica qual é o operador e marca as ocorrências.)

Ex: ‘+’ ‘-’ ‘\*’ ‘/’ ‘=’

Delimitadores Estruturais → (não guarda no banco, apenas identifica qual é o delimitador e marca as ocorrências.)  
Ex: ‘(‘ ’)’ ‘\[’ ‘\]’ ‘{’ ‘}’ ‘\\’ ‘/’ ‘|’ ‘’’ ‘“‘  
Símbolos Residuais → (não guarda no banco, apenas identifica qual é o Símbolo e marca as ocorrências.)   
‘%’ ‘$’ ‘@’ ‘\#’ ‘&’ ‘§’  
Emojis → (não guarda no banco, apenas identifica qual é o Emoji e marca as ocorrências.)   
Comandos Inline → (não guarda no banco, consulta na lista de comandos, apenas identifica qual é o comando, ativa, executa a sua função, desaparece.)   
Fórmulas → (não guarda no banco, consulta na lista de fórmulas, apenas identifica qual é o comando (usando as variáveis fornecidas), ativa, substitui o escrito pelo resultado do comando da fórmula, como no excel).   
};  
---

### **2.3 Ocorrência (instância no texto)**

* Instância localizada no buffer  
* Pode ser associada a:  
  * layers  
  * estruturas narrativas (novo)

struct Occurrence {  
   uint64\_t entity\_id;

   // NÃO depender exclusivamente de offsets  
   size\_t start\_offset;  
   size\_t end\_offset;

   uint32\_t flags;  
};

#### **Estrutura:**

Usar:

* GtkTextMarks (âncoras persistentes)  
* Estrutura tipo rope (avançado)

---

## **3\. Persistência Base**

Mantém:

* entities  
* occurrences  
* documents

---

## **4\. Processamento Incremental**

### **Fluxo:**

Usuário digita → GtkTextBuffer  
 ↓  
 Captura de mudança  
 ↓  
 Determinação de “dirty range”  
 ↓  
 Tokenização local  
 ↓  
 Normalização  
 ↓  
 Hash  
 ↓  
 Consulta/inserção em cache/banco  
 ↓  
 Atualização de ocorrências

### **Regras críticas:**

* Nunca reprocessar o documento inteiro  
* Ajustar offsets apenas na região afetada  
* Manter consistência eventual

---

## **5\. Sistema de Layers**

### **Modelo:**

* Layers pertencem às ocorrências  
* Relação N:N via tabela

**Integração Narrativa:**

* Pode ser usado como ferramenta auxiliar narrativa  
  * Ex: marcar “pensamento”, “foreshadowing”, etc.

### **Regra de visibilidade:**

Um trecho é visível se:

* Pertence a pelo menos 1 layer ativo

### **Override:**

* Possibilidade de ocultação forçada

### **Implementação:**

* `bitmask` usado apenas como cache de renderização  
* Fonte da verdade: tabela relacional

---

## **6\. Modelo Narrativo (Reestruturado)**

### **6.1 Conceito Central: Estrutura Narrativa**

Definição (formalizada):

Unidade narrativa delimitada no tempo que produz transformação significativa.

Essa unidade é independente de:

* documento  
* arquivo  
* bloco

---

### **6.2 Entidade Narrativa: Structure**

structures (  
   id INTEGER PRIMARY KEY,  
   name TEXT,  
   scale TEXT,  
   parent\_id INTEGER  
);

Campos:

* **name** → identificação  
* **scale** → (momento, cena, capítulo, arco…)  
* **parent\_id** → hierarquia

Correção:

* Substitui uso implícito de “documento como estrutura narrativa”

---

### **6.3 Hierarquia Narrativa**

Modelo:

* Estruturas são **recursivas**  
* Formam árvore:

Obra  
└── Saga  
     └── Volume  
          └── Arco  
               └── Capítulo  
                    └── Cena  
                         └── Momento  
---

### **6.4 Associação com Texto**

Nova tabela:

structure\_occurrences (  
   structure\_id INTEGER,  
   occurrence\_id INTEGER  
);

Regra:

* Estruturas **não armazenam texto**  
* Apenas **referenciam ocorrências**

Correção crítica:

* Mantém consistência com modelo baseado em buffer

---

## **7\. Banco de Dados (SQLite)**

### **Estratégia:**

* Uso de WAL mode  
* Escrita em batch (transações agrupadas)  
* Buffer intermediário em memória (write-behind)

---

### **Tabelas principais:**

CREATE TABLE entities (  
   id INTEGER PRIMARY KEY,  
   content TEXT UNIQUE,  
   type INTEGER  
);

CREATE TABLE occurrences (  
   id INTEGER PRIMARY KEY,  
   entity\_id INTEGER,  
   start\_offset INTEGER,  
   end\_offset INTEGER,  
   document\_id INTEGER  
);

CREATE TABLE documents (  
   id INTEGER PRIMARY KEY,  
   name TEXT  
);

CREATE TABLE layers (  
   id INTEGER PRIMARY KEY,  
   name TEXT  
);

CREATE TABLE occurrence\_layers (  
   occurrence\_id INTEGER,  
   layer\_id INTEGER  
);  
---

## **8\. Metadados Narrativos**

Sistema estruturado.

### **8.1 Modelo Geral**

narrative\_metadata (  
   id INTEGER PRIMARY KEY,  
   structure\_id INTEGER,  
   type TEXT,  
   content TEXT  
);

### **8.2 Função:**

* Direção narrativa  
* Roteiro  
* Intenção do autor

Observação:

* Metadados sempre ligados a uma **estrutura narrativa**, não diretamente ao documento

---

### **8.2 Tipos de Metadados**

Baseado no arquivo fornecido, os tipos são formalizados:

#### **1\. Propósito**

* objetivo  
* função  
* tema  
* valor de entrada/saída

#### **2\. Forma**

* POV  
* ritmo  
* linguagem  
* tom

#### **3\. Conflito**

* conflito central  
* tensão  
* ponto de virada  
* causa e efeito

#### **4\. Informação**

* o que o leitor sabe  
* ocultação  
* entrega de informação

#### **5\. Entidades Narrativas**

* agentes  
* papéis  
* motivações  
* estados  
* ações

#### **6\. Ambiente**

* tempo  
* espaço  
* clima  
* influência no personagem

#### **7\. Conflito Profundo**

* forças antagônicas  
* stakes  
* ferida  
* mentira  
* pacto narrativo

#### **8\. Tema**

* eixo de valores  
* gatilho  
* veredito

#### **9\. Premissa**

* quem  
* como  
* onde  
* por quê

---

### **8.3 Estrutura Formal vs Texto Livre**

Correção importante:

* Metadados podem ser:  
  * estruturados (campos definidos)  
  * texto livre (anotações)

---

## **9\. Workspaces**

CREATE TABLE workspaces (  
   id INTEGER PRIMARY KEY,  
   name TEXT  
);

### **Relações:**

* Workspace → múltiplos documentos  
* Documentos → blocos reutilizáveis

---

## **10\. Blocos de Texto**

Blocos deixam de ser “conteúdo” e passam a ser:

text\_blocks (  
   id INTEGER PRIMARY KEY,  
   structure\_id INTEGER,  
   version INTEGER  
);

Função:

* Representar uma **estrutura renderizável**

Observação:

* Bloco \= projeção de uma estrutura, não texto independente

---

## **11\. Sistema de Renderização**

## Possui dois eixos:

### **Entrada:**

* Lista ordenada de documentos/blocos  
* estruturas narrativas  
* ou documentos

### **Processo:**

* Resolver ocorrências  
* Aplicar layers  
* Concatenar visualmente  
* Respeitar ordem estrutural

### **Saída:**

* Preview (GTK)  
* Exportação

### **Otimização obrigatória:**

* Cache de renderização por:  
  * bloco  
  * combinação de layers ativos

**Renderização pode seguir:**

* ordem textual  
* ordem narrativa (hierárquica)

---

## **12\. Exportação / Compilação**

### **Permite dois modos:**

1. ### **Modo Texto**

   * ### baseado no buffer

2. ### **Modo Narrativo**

   * ### baseado em estruturas

### **Pipeline:**

1. Seleção de blocos  
2. Ordenação  
3. Aplicação de layers  
4. Geração final

### **Formatos:**

* TXT  
* Markdown  
* HTML  
* PDF (biblioteca externa)  
* EPUB (biblioteca externa)  
* DOCX (biblioteca externa)

---

## **13\. Corretor Ortográfico**

Integração com Hunspell.

### **Regras:**

* Aplicado apenas a `EntityType::Word`  
* Cache por hash

### **Correção:**

Invalidar cache quando:

* Regra de normalização mudar

---

## **14\. Consistência Global**

Separação final:

| Camada | Fonte da Verdade |
| :---- | :---- |
| Texto | GtkTextBuffer |
| Semântica | Entidades \+ Ocorrências |
| Narrativa | Estruturas \+ Metadados |

Regra central:

Renderização é prioritária. Processamento é eventual.

Isso implica:

* UI sempre responsiva  
* Atualizações progressivas  
* Possíveis estados intermediários consistentes

---

## **15\. Undo / Redo**

Cobre três níveis:

1. Texto  
2. Semântico  
3. Narrativo (estruturas e metadados)

Estratégia sugerida:

* Log de operações (event sourcing simplificado)  
* Reaplicação incremental

---

## **16\. Principais Desafios Técnicos**

* Sincronização buffer ↔ ocorrências  
* Definição correta de dirty ranges  
* Crescimento de memória (ocorrências)  
* Colisões de hash  
* Escrita intensiva no SQLite  
* Undo/Redo semântico  
* Coerência de versões de blocos

---

## **17\. Stack Tecnológica**

* C++20  
* GTK4  
* SQLite3  
* Meson  
* Threading: STL (`std::thread`, filas lock-free)  
* Spellcheck: Hunspell

---

## **18\. Estratégia de Implementação**

---

# **Classes C++**

# **1\. Núcleo: Tipos Fundamentais**

enum class EntityType {  
   Word,  
   Number,  
   Space,  
   Punctuation,  
   Operator,  
   Delimiter,  
   Symbol,  
   Emoji,  
   Command,  
   Formula  
};

using EntityId \= uint64\_t;  
using StructureId \= uint64\_t;  
using OccurrenceId \= uint64\_t;  
using DocumentId \= uint64\_t;  
---

# **2\. Camada Semântica**

## **2.1 Entity (imutável)**

class Entity {  
public:  
   Entity(EntityId id, std::string normalized, EntityType type);

   EntityId id() const;  
   const std::string& content() const;  
   EntityType type() const;

private:  
   EntityId id\_;  
   std::string content\_;  
   EntityType type\_;  
};

Responsabilidade:

* representar unidade semântica normalizada  
* nunca muda após criação

---

## **2.2 Occurrence (âncora no texto)**

class TextAnchor; // wrapper para GtkTextMark

class Occurrence {  
public:  
   Occurrence(OccurrenceId id, EntityId entity);

   OccurrenceId id() const;  
   EntityId entity() const;

   TextAnchor\* start();  
   TextAnchor\* end();

   void setFlags(uint32\_t flags);  
   uint32\_t flags() const;

private:  
   OccurrenceId id\_;  
   EntityId entity\_;

   TextAnchor\* start\_;  
   TextAnchor\* end\_;

   uint32\_t flags\_;  
};

Responsabilidade:

* representar uma instância no texto  
* manter posição estável via âncoras

---

## **2.3 EntityRegistry**

class EntityRegistry {  
public:  
   EntityId resolveOrCreate(const std::string& normalized, EntityType type);

   const Entity\* get(EntityId id) const;

private:  
   std::unordered\_map\<EntityId, Entity\> entities\_;  
   std::unordered\_map\<std::string, EntityId\> index\_;  
};

Responsabilidade:

* deduplicação  
* resolução por conteúdo

---

## **2.4 OccurrenceManager**

class OccurrenceManager {  
public:  
   OccurrenceId create(EntityId entity, TextAnchor\* start, TextAnchor\* end);

   Occurrence\* get(OccurrenceId id);

   void remove(OccurrenceId id);

private:  
   std::unordered\_map\<OccurrenceId, Occurrence\> occurrences\_;  
};  
---

# **3\. Camada Narrativa**

## **3.1 Structure**

class Structure {  
public:  
   Structure(StructureId id, std::string name, std::string scale);

   StructureId id() const;

   const std::string& name() const;  
   const std::string& scale() const;

   void setParent(StructureId parent);  
   StructureId parent() const;

   const std::vector\<StructureId\>& children() const;  
   void addChild(StructureId child);

private:  
   StructureId id\_;

   std::string name\_;  
   std::string scale\_;

   StructureId parent\_;  
   std::vector\<StructureId\> children\_;  
};

Responsabilidade:

* nó hierárquico narrativo  
* não contém texto

---

## **3.2 StructureGraph**

class StructureGraph {  
public:  
   StructureId create(const std::string& name, const std::string& scale);

   Structure\* get(StructureId id);

   void setParent(StructureId child, StructureId parent);

   const std::vector\<StructureId\>& roots() const;

private:  
   std::unordered\_map\<StructureId, Structure\> structures\_;  
   std::vector\<StructureId\> roots\_;  
};

Responsabilidade:

* gerenciar hierarquia  
* permitir múltiplas raízes (estruturas paralelas)

---

## **3.3 Associação Estrutura ↔ Ocorrência**

class StructureOccurrenceIndex {  
public:  
   void attach(StructureId structure, OccurrenceId occurrence);  
   void detach(StructureId structure, OccurrenceId occurrence);

   const std::vector\<OccurrenceId\>& occurrences(StructureId structure) const;  
   const std::vector\<StructureId\>& structures(OccurrenceId occurrence) const;

private:  
   std::unordered\_map\<StructureId, std::vector\<OccurrenceId\>\> s\_to\_o\_;  
   std::unordered\_map\<OccurrenceId, std::vector\<StructureId\>\> o\_to\_s\_;  
};

Responsabilidade:

* relação N:N  
* núcleo da ligação narrativa

---

## **3.4 NarrativeMetadata**

enum class MetadataType {  
   Purpose,  
   Form,  
   Conflict,  
   Information,  
   Entities,  
   Environment,  
   DeepConflict,  
   Theme,  
   Premise  
};

class NarrativeMetadata {  
public:  
   NarrativeMetadata(StructureId structure, MetadataType type, std::string content);

   StructureId structure() const;  
   MetadataType type() const;  
   const std::string& content() const;

private:  
   StructureId structure\_;  
   MetadataType type\_;  
   std::string content\_;  
};  
---

## **3.5 MetadataManager**

class MetadataManager {  
public:  
   void add(const NarrativeMetadata& data);

   std::vector\<NarrativeMetadata\> getByStructure(StructureId id) const;

private:  
   std::unordered\_multimap\<StructureId, NarrativeMetadata\> storage\_;  
};  
---

# **4\. Layers (Semântico)**

using LayerId \= uint64\_t;

class LayerManager {  
public:  
   LayerId create(const std::string& name);

   void attach(LayerId layer, OccurrenceId occurrence);  
   void detach(LayerId layer, OccurrenceId occurrence);

   const std::vector\<OccurrenceId\>& occurrences(LayerId layer) const;

private:  
   std::unordered\_map\<LayerId, std::string\> layers\_;  
   std::unordered\_map\<LayerId, std::vector\<OccurrenceId\>\> index\_;  
};  
---

# **5\. Documento / Texto**

## **5.1 Document**

class Document {  
public:  
   Document(DocumentId id, std::string name);

   DocumentId id() const;  
   const std::string& name() const;

private:  
   DocumentId id\_;  
   std::string name\_;  
};  
---

## **5.2 TextBufferAdapter (GTK)**

class TextBufferAdapter {  
public:  
   void insert(const std::string& text);  
   void erase(size\_t start, size\_t end);

   // integração com GtkTextBuffer  
};

Responsabilidade:

* encapsular GTK  
* gerar eventos de alteração

---

# **6\. Processamento**

## **6.1 Tokenizer / Normalizer**

class Tokenizer {  
public:  
   std::vector\<std::string\> tokenize(const std::string& text);  
};

class Normalizer {  
public:  
   std::string normalize(const std::string& token);  
};  
---

## **6.2 IncrementalProcessor**

class IncrementalProcessor {  
public:  
   void processRange(size\_t start, size\_t end);

private:  
   EntityRegistry\* entities\_;  
   OccurrenceManager\* occurrences\_;  
};

Responsabilidade:

* processar dirty ranges  
* atualizar ocorrências

---

# **7\. Renderização**

class Renderer {  
public:  
   std::string renderStructure(StructureId id);

private:  
   StructureGraph\* structures\_;  
   StructureOccurrenceIndex\* index\_;  
   OccurrenceManager\* occurrences\_;  
};

Comportamento:

* resolve ocorrências  
* ordena  
* gera saída

---

# **8\. Orquestrador Principal**

class SemanticTextEngine {  
public:  
   EntityRegistry entities;  
   OccurrenceManager occurrences;

   StructureGraph structures;  
   StructureOccurrenceIndex structureIndex;

   MetadataManager metadata;  
   LayerManager layers;

   Renderer renderer;  
};

Responsabilidade:

* ponto central do sistema  
* coordena todas as camadas

---

# **9\. Propriedades Emergentes do Modelo**

Esse design garante:

* texto único (GtkTextBuffer)  
* semântica reutilizável (entities)  
* narrativa sobreposta (structures)  
* múltiplas interpretações simultâneas  
* edição independente de organização narrativa

---

# **10\. Limitações (intencionais)**

* Estruturas não editam texto  
* Narrativa não depende de parsing completo  
* Sem ordem global obrigatória (exceto se definida por estrutura central)

