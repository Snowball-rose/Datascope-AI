# DataScope AI

A local-first desktop application for tabular data intelligence. Point it at a CSV or XLSX file and
it profiles the schema, scores data quality, previews cleaning steps, runs machine-learning
experiments, detects anomalies, replays rows through a concurrent streaming pipeline, and visualises
everything in 2D and 3D.

Everything runs on your machine. The original file is never modified — cleaning works on a working
copy that can be reset or exported at any time.

---

## 1. Quick start

### Run the bundled jar

```bash
java -jar datascope-ai.jar
```

Then click **Load demo data** on the Overview page, or drag a `.csv` / `.xlsx` file onto the window.
Requires **Java 21 or newer**.

### Build from source

```bash
mvn package                     # runs the tests, produces target/datascope-ai-1.0.0.jar
java -jar target/datascope-ai-1.0.0.jar
```

### Tests only

```bash
mvn test
```

---

## 2. The interface uses both JavaFX and Swing

This is a deliberate choice, not an accident of history. The two toolkits are good at different
things, so each owns the part of the application it does best, and they are composed in a single
window.

| Part of the application | Toolkit | Why |
|---|---|---|
| Application shell — sidebar, top bar, status bar, page switching | **JavaFX** | CSS theming, layout panes and transitions are far ahead of Swing's look-and-feel system |
| Dark theme (`datascope.css`) | **JavaFX** | One stylesheet themes the entire scene, including hover and focus states |
| Bar, line and histogram charts | **JavaFX** | Animated, CSS-styled charts with built-in axes, legends and layout |
| 3D point cloud (Explorer) | **JavaFX** | A real scene graph with a perspective camera and depth sorting, which Swing has no equivalent for |
| Page fade transitions, toasts, floating chart tooltips | **JavaFX** | `FadeTransition`, `PauseTransition` and `Tooltip` are declarative and GPU-friendly |
| **Dense data grids** — Overview preview, Profile schema, Clean change list, Anomalies flagged rows, Benchmark measurements | **Swing** | `JTable` with `TableRowSorter` provides sorting and filtering while paging keeps the Overview viewport small |
| Column-specific filtering with match modes | **Swing + JavaFX** | Search scans the full dataset off the UI thread; only the selected page is sent to the table |

The bridge is `javafx.embed.swing.SwingNode`: the Swing grids are real, live `JTable` components
hosted directly inside the JavaFX scene graph, sharing the same window and the same event loop.

So the grids are not a JavaFX table styled to look like Swing, and they are not a Swing table
awkwardly bolted on the side — they are genuine Swing components inside the JavaFX layout, and they
are themed to be visually indistinguishable from the JavaFX cards around them (see
`com.datascope.swing.SwingTheme`, which mirrors the CSS palette).

**The analytical engine knows about none of this.** The packages `model`, `io`, `analysis`, `ml` and
`stream` contain zero JavaFX and zero Swing imports, which is what makes it possible to drive them
from either toolkit.

---

## 3. Pages

| Page | What it does |
|---|---|
| **Overview** | Drop zone, KPIs, explainable quality score with a component breakdown, column chips, full-dataset filtering, and a sortable 500-row page. Double-click a row for every field. |
| **Profile** | Per-column schema: type, missing count and share, unique count, mean, min, max, standard deviation and a sample value. Select a column for a histogram (numeric) or a top-value chart (categorical). |
| **Insights** | Rule-based findings — missingness, constant and identifier columns, duplicates, skew — plus a correlation heatmap with per-cell hover showing *r* and its strength. |
| **Clean** | Preview-first cleaning: impute, clip outliers, drop rows, de-duplicate. Operations cover every row; all change details are browsable in 500-row preview pages. Export the working copy to CSV. |
| **ML Studio** | K-Means, PCA, logistic regression and linear regression, each with a held-out test score. Every run publishes a projection and its charts to the Explorer. |
| **Anomalies** | Robust median/MAD scan with a sensitivity slider, top-scores chart, flagged-row grid with the driving feature, CSV export, and a hand-off to the Explorer. |
| **Explorer** | Interactive 3D point cloud (drag to rotate, scroll to zoom, click a point to trace its source row) beside technique-specific charts. |
| **Live Stream** | Producer/consumer replay through a bounded queue with backpressure, throughput and alert telemetry, and a live signal you can hover for row, value and z-score. |
| **Benchmark** | Times K-Means, PCA, anomaly scoring and silhouette at increasing row counts, with a scaling curve and a normalised `ms / 1k rows` column. |
| **About** | What the app does, the technology behind it, and the available interactions. |

### Large datasets

CSV rows are read incrementally; XLSX sheets are parsed with Apache POI's SAX event reader. Imported
rows and cleaned working copies are stored in an indexed temporary file, so loading does not retain
all text cells in heap. The Overview page loads 500 rows at a time, searches the complete dataset
asynchronously, and anomaly results can be browsed/exported across all flagged rows.

Profiling, cleaning, ML fitting, anomaly detection, and Explorer projections use the complete loaded
dataset. Explorer rasterizes the full point cloud into the plot rather than creating one UI node per
row; dense areas blend point colors, and visible pixels remain clickable for source-row inspection.
Quadratic silhouette scoring and the K-Means elbow diagnostic use representative samples to keep
those diagnostics practical. Numeric ML operations still require a dense matrix for the selected
numeric features, and all-row projections use additional memory, so usage grows with
`rows × selected features` and remains bounded by the machine's available RAM.

### Keyboard

* `Ctrl+O` — open a file
* `Ctrl+1` … `Ctrl+9` — jump to a page
* Drag a `.csv` / `.xlsx` onto the window to load it

---

## 4. Project layout

```
datascope-ai/
├─ pom.xml                     Maven build (Java 21, JavaFX, Swing, POI, JUnit, Shade)
├─ README.md
├─ data/                       sample dataset (.csv and .xlsx)
└─ src/
   ├─ main/
   │  ├─ java/com/datascope/
   │  │  ├─ model/             DataSet, ColumnProfile, ColumnType        (no UI imports)
   │  │  ├─ io/                DataLoader — CSV + XLSX via POI           (no UI imports)
   │  │  ├─ analysis/          Profiler, DataQuality, Insights, Cleaner  (no UI imports)
   │  │  ├─ ml/                Stats, KMeans, PCA, Logistic/Linear,
   │  │  │                     AnomalyDetector, Metrics                  (no UI imports)
   │  │  ├─ stream/            ReplayEngine — bounded-queue replay       (no UI imports)
   │  │  ├─ swing/             ── Swing components hosted in JavaFX ──
   │  │  │  ├─ DataGrid.java   themed, sortable, filterable JTable + SwingNode wrapper
   │  │  │  └─ SwingTheme.java palette mirroring datascope.css
   │  │  ├─ app/               ── JavaFX application ──
   │  │  │  ├─ DataScopeApp    shell: sidebar, top bar, status bar, drag-and-drop
   │  │  │  ├─ AppState        observable shared state + Projection
   │  │  │  └─ views/          the ten page panels
   │  │  └─ util/              Ui (layout factories + spacing scale), Async (worker → FX thread)
   │  └─ resources/
   │     ├─ datascope.css      the dark theme
   │     └─ sample_customer_data.csv
   └─ test/java/com/datascope/ Cleaner, DataQuality, ML, Metrics/Stream tests
```

### Spacing

Every page is laid out from one spacing scale declared in `com.datascope.util.Ui`
(`GAP_BLOCK`, `GAP_CARD`, `GAP_ROW`, `GAP_TIGHT`, `PAD_PAGE`), and the matching card padding lives in
`datascope.css`. Views compose blocks rather than setting their own gaps, so the vertical rhythm is
identical across the application.

---

## 5. Algorithms — all implemented from scratch

* **Column type inference** — numeric / categorical / boolean / datetime / text, by sampling.
* **Data-quality score** — a weighted blend of completeness (40%), uniqueness (20%), consistency
  (15%), validity (10%) and usefulness (15%), reported component by component.
* **K-Means** with k-means++ seeding, inertia reporting and an elbow curve.
* **PCA** by power iteration with deflation (scree and cumulative explained variance).
* **Logistic regression** (sigmoid, gradient descent) and **linear regression**.
* **Robust anomaly scoring** on median / MAD, so an extreme value cannot mask itself.
* **Evaluation** — train/test split, confusion matrix, precision / recall / F1, R², RMSE, silhouette.

---

## 6. Notes

* All heavy work — loading, model fitting, benchmarks, anomaly scans — runs off the UI thread and
  returns through `SwingUtilities.invokeLater` or `Platform.runLater`, so the interface stays
  responsive.
* The application starts empty by design; the sample dataset loads only when you ask for it.
* Charts, the heatmap and the 3D cloud all support hover readouts; the Explorer additionally traces
  any point back to its original source row.

## 7. Sample dataset

`data/sample_customer_data.csv` — 1,500 customer records × 10 columns (age, income, visits,
purchases, cart value, support tickets, tenure, churn, plan, city), with some intentionally missing
values to exercise the cleaning and quality features.
