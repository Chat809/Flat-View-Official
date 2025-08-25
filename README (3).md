# Pipeline de Autenticação de Imagens Ambientais
> **Tema:** identificação de *deepfake geography* e verificação de imagens ambientais com ênfase em **solo exposto / desmatamento** a partir de imagens de satélite.

Este repositório contém uma *pipeline* simples, transparente e explicável construída para o Google Colab.  
Ela realiza **segmentação baseada em cor** (técnicas clássicas de Visão Computacional) para destacar **solo exposto visível** (tons terrosos) e **quantificar** a área, gerando um **diagnóstico objetivo** (baixa/moderada/alta suspeita de desmatamento).

---

## ✨ Objetivos
- **Autenticar** e verificar imagens ambientais contra manipulações visuais (deepfake geography).
- **Detectar** padrões **reais** de solo exposto (marrom/amarelado), ignorando vegetação, água, nuvem e fumaça.
- **Explicar** claramente o processo com máscaras e *overlays* (resultados interpretáveis).

---

## 🗂️ Dados
- **Fonte:** [Kaggle – Wildfire Prediction Dataset (Satellite Images)](https://www.kaggle.com/)  
  *(você informará o slug exato no Colab; ex.: `abdelghaniaaba/wildfire-prediction-dataset`)*
- **Formato:** `.jpg`, `.png`, `.tif` (canais RGB).  
- **Observação:** A pipeline funciona com bandas visíveis; para NDVI/NBR (multibanda com NIR), há notas na seção “Extensões”.

---

## 🧰 Ferramentas, Dependências e Bibliotecas
- **Google Colab** (ambiente recomendado).
- **Kaggle API**: download e listagem de datasets.
- **Python 3.10+**
- **Bibliotecas Python:**
  - `opencv-python` (OpenCV) – leitura/gravura de imagens, conversões de cor, operações morfológicas.
  - `numpy` – vetorização e aritmética de matrizes.
  - `matplotlib` – visualização (plots e *overlays*).
  - `scikit-learn` – `KMeans` (quando utilizado).
  - `tqdm` – barra de progresso.
  - (opcional) `zipfile`, `glob`, `os`, `json`, `shutil` – utilitários de E/S.

> Instalação mínima no Colab (já incluída na 1ª célula):
```bash
!pip -q install kaggle opencv-python scikit-learn matplotlib numpy tqdm
```

---

## 🔐 Autenticação Kaggle
1. Crie/acesse sua conta no **Kaggle** → **My Account** → **Create New API Token**.
2. Baixe o arquivo `kaggle.json`.
3. Na 1ª célula do Colab, **faça upload** do `kaggle.json`; o script salva em `/root/.kaggle/kaggle.json` (permissão `600`).

---

## ⚙️ Parâmetros (células finais)
| Parâmetro | Função |
|---|---|
| `K_CLUSTERS` | Nº de grupos no K-means (quando usado). |
| `MAX_SIDE` | Redimensionamento máximo do lado da imagem (velocidade x detalhe). |
| `ALPHA` | Transparência do *overlay* laranja. |
| `GREEN_EXG` | Limiar para rejeitar vegetação (Excesso de Verde). |
| `WATER_B_DEL` | Limiar para rejeitar água (azul muito dominante). |
| `SMOKE_S_MAX` | Saturação baixa para detectar fumaça/cinza. |
| `SMOKE_V_MIN` | Brilho mínimo para fumaça/cinza. |
| `OTSUSTRETCH` | Expansão do limiar de Otsu (↑ aumenta a máscara). |
| `MIN_AREA_FRAC` | Remove componentes muito pequenos (fração da imagem). |
| `DILATE_ITERS` | Dilatação pós-processamento (↑ expande fronteiras). |

---

## 🧪 Estrutura da Pipeline (7 células principais)

### **1) Setup do ambiente**
- Upload do `kaggle.json` + configuração de permissões.
- Instalação de dependências (OpenCV, NumPy, Matplotlib, scikit-learn, tqdm).
- **Objetivo:** deixar Colab pronto para baixar e processar dados.

### **2) Busca de datasets (Kaggle)**
- Uso do comando: `!kaggle datasets list -s "<termo>"` (ex.: “wildfire”, “burned area”).
- **Objetivo:** localizar e escolher a base (copiar o *slug*).

### **3) Download & Extração**
- Define `DATASET_SLUG` (ex.: `abdelghaniaaba/wildfire-prediction-dataset`).
- `!kaggle datasets download -d {DATASET_SLUG} -p /content/data_raw --force`
- Extrai `.zip` para `/content/data` e lista os arquivos.
- **Objetivo:** disponibilizar as imagens localmente no Colab.

### **4) Descoberta de imagens**
- Varredura recursiva com `glob` para (`*.jpg`, `*.jpeg`, `*.png`, `*.tif`, `*.tiff`).
- **Objetivo:** conferir contagem e caminhos das imagens.

### **5) Segmentação em Lote (solo exposto visível)**
**Conceito:** Pontuação espectral por pixel + limiar adaptativo (Otsu) → máscara binária → *overlay*.

- **Pré-processamento:** `GaussianBlur` leve; redimensionamento (`MAX_SIDE`).
- **Canais de cor usados:** 
  - **HSV**: preferência por H~10–45 (laranja/terroso), S e V **médios** (evita nuvem/branco e sombra/água).
  - **Lab**: `a*>0` e `b*>0` (puxa para vermelho e amarelo).
  - **RGB**: “solo visível” tende a ter **R > G,B**; usa-se um *índice visível de solo* (VBSI aproximado).
- **Supressões (zeradas):** vegetação (ExG alto), água (azulado dominante + V baixo), nuvem (V alto & S baixo) e fumaça (S muito baixo & V médio).
- **Score final:** combinação ponderada de HSV + Lab + VBSI → normalização robusta (percentis) → **Otsu** para binarizar.
- **Pós-processamento:** remoção de ilhas pequenas (`MIN_AREA_FRAC`), **closing** (preenche buracos) e **dilatação** opcional.
- **Saídas (por imagem):** 
  - `*_mask_brown.png` — máscara binária (solo exposto visível),  
  - `*_overlay_brown.jpg` — *overlay* laranja sobre a imagem original.

### **6) Classificação (imagem única)**
**Conceito:** transformar a máscara em um **indicador** (% de solo exposto) + **status**.

- Upload de uma imagem → calcula a máscara (mesma lógica da etapa 5).
- **Métrica:** `% de pixels = 255` na máscara.
- **Regras padrão:** 
  - `< 5%` → **Baixa suspeita**; 
  - `5–20%` → **Suspeita moderada**; 
  - `≥ 20%` → **Alta suspeita**.
- **Saída:** figura com **Imagem | Overlay | Máscara** + texto com o status e o percentual.

### **7) Teste manual / Exportação (opcional)**
- Rodar a etapa 6 diversas vezes com imagens distintas.
- (Opcional) Compactar pastas de saída:
```python
import shutil
shutil.make_archive("/content/outputs_brown", "zip", "/content/outputs")
```
- **Objetivo:** avaliação manual rápida e compartilhamento dos resultados.

---

## ▶️ Como Executar (resumo)
1. Abra o **Google Colab** e cole as células na ordem (1 → 7).  
2. **Faça upload** do `kaggle.json` na 1ª célula e execute-a.  
3. Na célula 2, **liste datasets** e escolha o *slug* desejado.  
4. Na célula 3, **baixe e extraia** os dados.  
5. Na célula 4, **confira** as imagens.  
6. Na célula 5, **processe em lote** (gerará as máscaras e *overlays* em `/content/outputs`).  
7. Na célula 6, **teste com upload manual** e verifique o **status de suspeita**.

---

## 📊 Interpretação de Resultados
- **Máscara** (`*_mask_brown.png`): branco = **solo exposto visível**; preto = descartado (vegetação, água, nuvem/fumaça, sombra).
- **Overlay** (`*_overlay_brown.jpg`): áreas marcadas em **laranja**.
- **Percentual**: fração da imagem classificada como solo exposto → **indicador** de (baixa/moderada/alta) suspeita de desmatamento.
- **Atenção:** a pipeline **não atravessa nuvem densa** (apenas solo visível).

---

## ⚠️ Limitações
- Heurística baseada em cor; pode sofrer em cenários **muito iluminados/sombreados** ou com **solos de cores atípicas**.
- **Nuvem densa** obstrui o dado óptico (não há como “ver por baixo”).
- Sem georreferenciamento/temporal: não mede área real (km²) nem monitora variação no tempo (extensões abaixo contornam isso).

---

## 🔭 Extensões Sugeridas
- **Bandas espectrais** (Sentinel-2/Landsat): usar **NDVI/NBR** e limiares físicos; separar **solo exposto x cicatriz de queimada**.
- **Georreferenciamento**: calcular área real (km²) a partir de metadados.
- **Modelos supervisionados**: treinar um classificador com rótulos (RF, SVM, CNN) e comparar com a heurística.

---

## 📎 Citação (exemplo)
> WT Sistemas (2025). *Pipeline de Autenticação de Imagens Ambientais: detecção de solo exposto visível com heurística de cor.* Google Colab.  

---

## 📝 Licença
Uso acadêmico/educacional. Ajuste conforme a política do seu curso/instituição.

