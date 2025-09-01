---
title: Flujo de trabajo

---

# Pre-procesamiento de las muestra

Las secuencias de gran tamaño (>5 G) depositadas en el NCBI se pueden descargar desde ENA, sin necesidad de usar SRA Toolkit. 

El Bioproyecto [PRJNA254808](https://www.ebi.ac.uk/ena/browser/view/PRJNA254808) contiene todas muestras de la estación 6 y 10.
El Bioproyecto [PRJNA632347](https://www.ebi.ac.uk/ena/browser/view/PRJNA632347) contiene muestras de la estación 2 y 6.

Cada una de estas lecturas se ha guardado en un archivo de lectura de secuencias o *Sequence Read Archive* (SRA) con las claves de identificación:

~~~~
1. SRR1509790
1. SRR1509792
1. SRR1509793
1. SRR1509794
1. SRR1509796
1. SRR1509797
1. SRR1509798
1. SRR1509799
1. SRR1509800
~~~~

---

## Descarga en conjunto de las muestras

Consiste en recuperar el script de descarga que genera la misma plataforma. Este script se descarga en el equipo de computo personal y porteriormente se sube a la cuenta del servidor del CIBNOR con ayuda del programa [Cyberduck](https://cyberduck.io/).



---
Antes de iniciar con la descarga en el servidor del CIBNOR, se puede ejecutar el siguiente comando para verificar el contenido del script proporcionado por ENA:

```
$ cat ena-file-download-selected-files-[....].sh
```
Con este comando se pueden visualizar los links de descarga para cada SRA del bioproyecto.

Para ejecutar el script, primero se debe proporcionar el permiso con:

```
$ chmod +x ena-file-download-selected-files-[....].sh
```

Una vez realizado lo anterior, se puede ejecutar el script con el siguiente comando para dejarlo corriendo, poder siguir utilizando la terminal y evitar perder el progreso aunque se cierre la terminal.

```
$ nohup ./ena-file-download-selected-files-20250529-0312.sh > out_salida_st10.txt 2>&1 &
```
Para checar el progreso o algún error que se presente se puede consultar el archivo `out_salida_st10.txt`:

```
$ cat out_salida_st10.txt
```

O bien, solo las últimas líneas del archivo:

```
$ tail -f out_salida_st10.txt
```

---
 ## Ejecución de FastQC
 
El programa FastQC, a grandes rasgos, permite conocer y analizar la calidad de las lecturas en una muestra. Genera un informe descriptivo con gráficos y estadísticos.

---

Se han descargado 29 muestras pared-end, dando un total de 58 archivos .fastq.gz que serán analizados uno por uno en FastQC de manera automatizada.

El primer paso es activar el ambiente de conda en el que se encuentra FastQC.

```
$ conda activate fastqc
```

En seguida, abrir y crear un archivo de texto con `nano` para escribir las indicaciones del script:

~~~~
#!/bin/bash

# Crear carpeta para los resultados si no existe
mkdir -p reads_fastqc

# Ejecutar FastQC en todos los archivos .fastq.gz del directorio
for file in *.fastq.gz; do
    echo "Procesando $file ..."
    fastqc "$file" --outdir reads_fastqc
done

echo "Análisis FastQC completado."
~~~~

Por último, proporcionar los permisos de ejecución y correr el script:

```
$ chmod +x run_fastqc.sh

$ nohup ./run_fastqc.sh > out_fastqc_untrimmed.txt  2>&1 &
```

Una vez que FastQC ha terminado de analizar los archivos, podemos descargar a nuestro equipo local los archivos `.html` con ayuda del programa Cyberduck y una vez descargados abrirlos en el navegador.

Otra opción es descomprimir los archivos `.zip` y ver los resultados del reporte, esta manera no es tan visual pero podemos identificar las pruebas que pasaron y fallaron en el análisis con FastQC.

```
$ for filename in *.zip
> do
> unzip $filename
> done
```

Primero se concatenan los archivos `summary.txt` de cada archivo en un solo documento para poder conocer con el comando `grep` el estado de la prueba.

```
$ cat */summary.txt > ~/directorio/fastqc_summaries.txt

$ grep FAIL fastqc_summaries.txt
```

Desactivamos el ambiente conda fastqc mediante:

```
$ conda deactivate
```

---
## Ejecución de MultiQC
A diferencia del programa FastQC, el programa MultiQC realiza un solo reporte con todos los archivos que queremos analizar. También es un reporte con gráficas y estadísticos que permite comparar la información contenida en cada archivo.

---
Antes de iniciar con la ejecución de MultiQC, activar el entorno conda donde se encuentra el programa:

```
$ conda activate metagenomas
```

Nuevamente seguir los pasos para crear un script con `nano`, otorgar los permisos con `chmod +x`  y ejecutar con `nohup` y `&`.

Indicaciones del script:

```
#!/bin/bash

echo "Iniciando análisis MultiQC..."

multiqc reads_fastqc --outdir reads_multiqc

echo "Análisis MultiQC completado."
```

Se ha originado un único archivo `.html` que puede ser descargado al equipo local con ayuda de Cyberduck para visualizar su contenido y tener una idea clara de los parámetros que te tomarán en el siguiente paso.

Desactivar el ambiente conda:

```
$ conda deactivate
```
---

## Ejecución de Trimmomatic

Trimmomatic es una herramienta que permite recortar y filtar aquellas lecturas de mejorar calidad. Es un paso escencial para determinar los resultados de la asignación taxonómica y/o el ensamblaje.


---


Se consideraron los siguientes parámetros reportados en [Alexander, et al 2023](https://journals.asm.org/doi/10.1128/mbio.01676-23):

~~~~
ILLUMINACLIP:TruSeq3-PE.fa:2:30:7 \
    LEADING:2 \
    TRAILING:2 \
    SLIDINGWINDOW:4:2 \
    MINLEN:50
~~~~

Antes de ejecutar Trimmomatic asegurarse de contar con el archivo `TruSeq3-PE.fa` o en dado caso descargarlo con `wget` de la página de GitHub https://raw.githubusercontent.com/timflutre/trimmomatic/master/adapters/TruSeq3-PE.fa 

El script para Trimommatic:
```
#!/bin/bash

# Crear carpeta de salida si no existe
mkdir -p trimmed2_fastq

# Ruta al archivo de adaptadores
ADAPTERS="TruSeq3-PE.fa"

# Bucle sobre todos los archivos _R1.fastq.gz
for R1 in *_1.fastq.gz
do
  # Obtener el prefijo de la muestra (antes de _R1)
  PREFIX=$(basename "$R1" _1.fastq.gz)

  # Definir archivo R2 correspondiente
  R2="${PREFIX}_2.fastq.gz"

  # Definir archivos de salida
  OUT1_P="trimmed_fastq/${PREFIX}_1_paired.fastq.gz"
  OUT1_U="trimmed_fastq/${PREFIX}_1_unpaired.fastq.gz"
  OUT2_P="trimmed_fastq/${PREFIX}_2_paired.fastq.gz"
  OUT2_U="trimmed_fastq/${PREFIX}_2_unpaired.fastq.gz"

  echo "Procesando muestra: $PREFIX"

  # Ejecutar Trimmomatic con parámetros conservativos
  trimmomatic PE -threads 4 -phred33 \
    "$R1" "$R2" \
    "$OUT1_P" "$OUT1_U" \
    "$OUT2_P" "$OUT2_U" \
    ILLUMINACLIP:${ADAPTERS}:2:30:7 \
    LEADING:2 \
    TRAILING:2 \
    SLIDINGWINDOW:4:2 \
    MINLEN:50

  echo "Finalizado: $PREFIX"
  echo "--------------------------------------------"
done

echo "Análisis con Trimommatic completado."
```


---
## Ejecución de MEGAHIT

MEGAHIT es un ensamblador NGS que está optimizado para metagenomas. Une fragmentos cortos o reads produciendo secuencias más largas y continuas llamadas contigs.


---
Para la ejecución de Megahit se han creado dos scripts con diferente longitud de conting. Esto permite comparar el número de conting reconstruidos a 1000 y 3000 pd.

El script será el siguiente:

```
#!/bin/bash

# Directorio de tus archivos FASTQ trimmed
TRIMMED_FASTQ_DIR="/home/sfonseca/sta_10/trimmed2_fastq" # Mantengo esta ruta como en el último script


# Directorio principal para los resultados de ensamblaje de MEGAHIT
MEGAHIT_OUT_DIR="/home/sfonseca/sta_10/assembled3_contigs" # ¡DIRECTORIO DE SALIDA MODIFICAR EN CADA CAMBIO DE MIN_CONTIG_LENGTH!


# Directorio para los archivos de registro (logs)
MEGAHIT_LOGS="/home/sfonseca/sta_10/assembly3_logs" 

# Crear directorios base si no existen
mkdir -p "$MEGAHIT_OUT_DIR"
mkdir -p "$MEGAHIT_LOGS"

# Parámetros de MEGAHIT, modificar en MIN_CONTIG_LENGTH la longitud con la que se desea trabajar, en este caso 1000 pd

MEGAHIT_K_MERS="21,41,61,81,101,121,141"
MIN_CONTIG_LENGTH="1000"
NUM_THREADS="4"


# Obtener prefijos de las muestras (ej. SRRXXXXXXX)
SAMPLES=$(ls "$TRIMMED_FASTQ_DIR"/*_1_paired.fastq.gz | xargs -n1 basename | sed 's/_1_paired.fastq.gz//')

# Bucle para cada muestra
for SAMPLE_PREFIX in $SAMPLES
do
echo "Iniciando ensamblaje para: $SAMPLE_PREFIX"

  # Rutas de entrada y salida para la muestra actual
  READ1="${TRIMMED_FASTQ_DIR}/${SAMPLE_PREFIX}_1_paired.fastq.gz"
  READ2="${TRIMMED_FASTQ_DIR}/${SAMPLE_PREFIX}_2_paired.fastq.gz"
  SAMPLE_OUTPUT_DIR="${MEGAHIT_OUT_DIR}/${SAMPLE_PREFIX}_assembly"
  SAMPLE_LOG_FILE="${MEGAHIT_LOGS}/${SAMPLE_PREFIX}_megahit.log"


  # Ejecutar MEGAHIT
  megahit \
    -1 "$READ1" \
    -2 "$READ2" \
    -o "$SAMPLE_OUTPUT_DIR" \
    --min-contig-len "$MIN_CONTIG_LENGTH" \
    --presets meta-sensitive \
    --k-list "$MEGAHIT_K_MERS" \
    -t "$NUM_THREADS" \
    > "$SAMPLE_LOG_FILE" 2>&1

  echo "Finalizado: $SAMPLE_PREFIX"
done

echo "Todos los ensamblajes han sido procesados."
```


---
## Ejecución de CCMetagen


---
