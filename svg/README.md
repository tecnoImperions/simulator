# 🖼️ Procesador de Imágenes — Documentación de Fórmulas

## ¿Cómo funciona el Canvas de imágenes?

Cuando cargamos una imagen en un `<canvas>` de HTML5, podemos acceder a cada **píxel** individualmente mediante `ImageData`. Cada píxel tiene **4 componentes** (canales) en orden:

```
[R, G, B, A,  R, G, B, A,  R, G, B, A, ...]
  píxel 0      píxel 1      píxel 2
```

| Canal | Rango | Significado              |
|-------|-------|--------------------------|
| R     | 0–255 | Rojo (Red)               |
| G     | 0–255 | Verde (Green)            |
| B     | 0–255 | Azul (Blue)              |
| A     | 0–255 | Opacidad (Alpha)         |

Para acceder al píxel en la posición `(x, y)` de una imagen de ancho `W`:

```
índice = (y × W + x) × 4
```

---

## Filtros implementados

### 1. 🟤 Sepia

Simula el envejecimiento fotográfico. Mezcla los canales RGB con pesos específicos para dar un tono cálido amarillo-marrón.

**Fórmulas:**
```
R_out = (R × 0.393) + (G × 0.769) + (B × 0.189)
G_out = (R × 0.349) + (G × 0.686) + (B × 0.168)
B_out = (R × 0.272) + (G × 0.534) + (B × 0.131)
```

> **¿Por qué estos coeficientes?**  
> Son una aproximación de la respuesta espectral del proceso químico del telurato de plata (sepía) usado en fotografía antigua. El rojo domina, el verde es intermedio y el azul queda suprimido.

**Clamp obligatorio:** los valores se limitan al rango [0, 255]:
```
R_out = min(255, R_out)
```

---

### 2. 🔄 Invertir (Negativo)

Invierte cada canal restándolo de 255. Es la operación de complemento a 255.

**Fórmula:**
```
R_out = 255 - R
G_out = 255 - G
B_out = 255 - B
```

> El canal Alpha **no se toca**. El negativo de un negativo es la imagen original.

**Caso especial:** blanco (255,255,255) → negro (0,0,0) y viceversa.

---

### 3. ⬛ Binario — Blanco y Negro (Grayscale)

Convierte a escala de grises usando la **fórmula de luminancia perceptual** (estándar ITU-R BT.709):

**Fórmula principal:**
```
L = 0.2126 × R + 0.7152 × G + 0.0722 × B
```

Luego se asigna esa luminancia a los tres canales:
```
R_out = G_out = B_out = L
```

> **¿Por qué no promediar?** El ojo humano es más sensible al verde (~72%) y menos al azul (~7%). Si promediáramos los tres por igual, el resultado se vería artificialmente brillante o apagado en ciertas fotos.

**Comparación de métodos:**

| Método        | Fórmula                        | Calidad |
|---------------|--------------------------------|---------|
| Promedio      | (R + G + B) / 3               | ❌ Pobre |
| Luminosidad   | 0.299R + 0.587G + 0.114B      | ✅ Buena (BT.601) |
| **Luminancia**| **0.2126R + 0.7152G + 0.0722B**| ✅✅ Mejor (BT.709) |

---

### 4. 🔁 Resetear

No aplica transformación matemática. Recarga los datos originales del `ImageData` que fueron guardados en memoria al momento de cargar la imagen.

```js
ctx.putImageData(originalImageData, 0, 0);
```

---

## Flujo del programa

```
[Usuario carga imagen PNG/JPG]
         ↓
[Se dibuja en <canvas> con drawImage()]
         ↓
[Se extrae ImageData con getImageData()]
         ↓
[Se guarda copia original en memoria]
         ↓
[Usuario elige filtro]
         ↓
[Se itera píxel por píxel en bucle]
         ↓
[Se aplica la fórmula al triplete R,G,B]
         ↓
[Se escribe el resultado con putImageData()]
         ↓
[Canvas actualizado en pantalla]
```

---

## Estadísticas de Color

Al procesar la imagen también se calculan:

- **Brillo promedio:** `Σ(L) / N_píxeles`  donde `L` es la luminancia de cada píxel
- **Canal dominante:** el de mayor suma acumulada entre R, G, B totales
- **Histograma:** distribución de luminancias en 32 cubetas (bins) de rango [0,255]

---

## Librerías utilizadas

| Librería    | Versión | Uso                                      |
|-------------|---------|------------------------------------------|
| `matter-js` | 0.19    | Motor de física (partículas animadas en fondo) |
| Canvas API  | nativo  | Procesamiento de píxeles                 |
| File API    | nativo  | Lectura de archivos PNG/JPG locales      |

---

## Notas de implementación

- El bucle de píxeles opera sobre un `Uint8ClampedArray`, que automáticamente satura valores fuera de [0,255].
- Para imágenes grandes (>4 MP) se recomienda usar `OffscreenCanvas` o `Worker` para no bloquear el hilo principal.
- La vista previa en miniatura usa `canvas.toDataURL('image/jpeg', 0.7)` para eficiencia.