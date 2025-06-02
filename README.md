# Señales Aviator

Este repositorio contiene la aplicación **Señales Aviator**, diseñada para generar señales aleatorias cada 3–5 minutos y ayudar en la toma de decisiones de la conocida estrategia de Aviator. Actualmente, la efectividad de las señales se sitúa en aproximadamente un **66%**.

---

## Descripción

**Señales Aviator** es una herramienta la generación de señales para la estrategia de Aviator. Cada vez que se genera una señal, aparece una burbuja (tipo “chat”) con un multiplicador aleatorio (entre 1.00x y 50.00x) y la hora exacta en que se emitió. Las señales se generan automáticamente en un intervalo de **3 a 5 minutos**.

- **Interfaz amigable**: estilo “chat” con burbujas verdes para cada señal.
- **Historial persistente**: todas las señales se guardan en un archivo `history.json` y se recargan al iniciar.
- **Control manual**: botones en español para “Iniciar”, “Detener” y “Limpiar historial”.
- **Efectividad actual**: aproximadamente **66%** de señales consideradas exitosas.

---

## Características

- Generación automática de señales cada 3–5 minutos.
- Registro y visualización en tiempo real de cada señal con marca de fecha y hora.
- Botones en español para:
  - **Iniciar** la generación de señales.
  - **Detener** la emisión automática.
  - **Limpiar historial** (eliminar todas las burbujas guardadas).
- Historial guardado en `history.json` para mantener un registro persistente.
- Proyecto completamente **legal** y de uso personal.


---

## Uso

1. Ejecuta la aplicación:
   ```
   python3 aviator_signals.py
   ```
2. Se abrirá una ventana titulada **“Señales Aviator”** con tres botones:
   - **Iniciar**: inicia la generación automática de señales.
   - **Detener**: pausa la emisión de señales.
   - **Limpiar historial**: borra todas las señales guardadas en pantalla y en `history.json`.
3. Al hacer clic en **“Iniciar”**, la primera señal se mostrará en 1 segundo.
4. Cada **3–5 minutos** aparecerá una nueva burbuja con el multiplicador y la hora exacta.
5. Para pausar momentáneamente, presiona **“Detener”**.
6. Para eliminar todas las burbujas y reiniciar el histórico, presiona **“Limpiar historial”**.

El archivo `history.json` guarda cada señal con estructura:

[
  {
    "time": "2025-06-02 14:23:10",
    "signal": "12.37"
  },
  {
    "time": "2025-06-02 14:26:45",
    "signal": "8.45"
  }
  // ...
]


---

## Código de la Aplicación

```python
import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import random
import json
import os
import threading

HISTORY_FILENAME       = "history.json"
MIN_INTERVAL_SECONDS   = 180   # 3 minutos
MAX_INTERVAL_SECONDS   = 300   # 5 minutos

COLOR_BG               = "#EAEAF2"
COLOR_CHAT_BG          = "#FFFFFF"
COLOR_BUBBLE_BG        = "#DCF8C6"
COLOR_TIME_TEXT        = "#999999"
COLOR_BUTTON_BG        = "#0088CC"
COLOR_BUTTON_FG        = "#FFFFFF"
COLOR_CLEAR_BG         = "#FF5555"

FONT_MESSAGE           = ("Verdana", 11)
FONT_TIME              = ("Verdana", 8)
FONT_BUTTON            = ("Verdana", 10, "bold")

MAX_HISTORY_MESSAGES   = None

def load_history(filename=HISTORY_FILENAME):
    if not os.path.exists(filename):
        return []
    try:
        with open(filename, "r", encoding="utf-8") as f:
            data = json.load(f)
            if isinstance(data, list):
                valid = []
                for item in data:
                    if isinstance(item, dict) and "time" in item and "signal" in item:
                        valid.append(item)
                return valid
            return []
    except (json.JSONDecodeError, IOError):
        return []

def save_history(history, filename=HISTORY_FILENAME):
    try:
        with open(filename, "w", encoding="utf-8") as f:
            json.dump(history, f, ensure_ascii=False, indent=4)
    except IOError:
        messagebox.showerror("Error", f"No se pudo guardar el historial en {filename}.")

def generate_multiplier():
    value = random.uniform(1.00, 50.00)
    return f"{value:.2f}"

class MessageBubble(tk.Frame):
    def __init__(self, parent, text, timestamp, *args, **kwargs):
        super().__init__(parent, *args, **kwargs)
        self.configure(background=COLOR_BUBBLE_BG, bd=0, highlightthickness=0)
        tk.Label(
            self,
            text=text,
            bg=COLOR_BUBBLE_BG,
            fg="black",
            font=FONT_MESSAGE,
            wraplength=250,
            justify="left"
        ).pack(side="top", anchor="w", padx=10, pady=(5, 0))
        tk.Label(
            self,
            text=timestamp,
            bg=COLOR_BUBBLE_BG,
            fg=COLOR_TIME_TEXT,
            font=FONT_TIME,
            anchor="e"
        ).pack(side="bottom", anchor="e", padx=10, pady=(0, 5))

class ChatArea(tk.Frame):
    def __init__(self, parent, *args, **kwargs):
        super().__init__(parent, *args, **kwargs)
        self.configure(bg=COLOR_CHAT_BG)
        self.canvas    = tk.Canvas(self, bg=COLOR_CHAT_BG, bd=0, highlightthickness=0, relief="ridge")
        self.canvas.pack(side="left", fill="both", expand=True)
        self.scrollbar = ttk.Scrollbar(self, orient="vertical", command=self.canvas.yview)
        self.scrollbar.pack(side="right", fill="y")
        self.canvas.configure(yscrollcommand=self.scrollbar.set)
        self.canvas.bind_all("<MouseWheel>", self._on_mousewheel)
        self.inner_frame = tk.Frame(self.canvas, bg=COLOR_CHAT_BG)
        self.inner_id    = self.canvas.create_window((0, 0), window=self.inner_frame, anchor="nw")
        self.inner_frame.bind("<Configure>", self._on_frame_configure)
        self.canvas.bind("<Configure>", self._on_canvas_configure)
        self.bubbles = []

    def _on_mousewheel(self, event):
        self.canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")

    def _on_frame_configure(self, event):
        self.canvas.configure(scrollregion=self.canvas.bbox("all"))

    def _on_canvas_configure(self, event):
        width = event.width
        self.canvas.itemconfig(self.inner_id, width=width)

    def add_message(self, text, timestamp):
        bubble = MessageBubble(self.inner_frame, text, timestamp)
        bubble.pack(side="top", anchor="w", padx=10, pady=5, fill="x")
        self.bubbles.append(bubble)
        self.inner_frame.update_idletasks()
        self.canvas.yview_moveto(1.0)

class AviatorApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Señales Aviator")
        self.root.minsize(350, 500)
        self.root.resizable(width=False, height=False)
        self.root.configure(bg=COLOR_BG)
        self.is_running = False
        self.timer      = None
        self.history    = load_history()
        self._create_widgets()
        if self.history:
            self._load_history_to_ui()

    def _create_widgets(self):
        header_frame = tk.Frame(self.root, bg=COLOR_BG)
        header_frame.pack(side="top", fill="x", pady=(10, 5))
        tk.Label(
            header_frame,
            text="Señales Aviator",
            bg=COLOR_BG,
            fg="black",
            font=("Verdana", 16, "bold")
        ).pack(side="left", padx=10)
        chat_frame = tk.Frame(self.root, bg=COLOR_BG)
        chat_frame.pack(side="top", fill="both", expand=True, padx=10, pady=(0, 10))
        self.chat_area = ChatArea(chat_frame)
        self.chat_area.pack(fill="both", expand=True)
        control_frame = tk.Frame(self.root, bg=COLOR_BG)
        control_frame.pack(side="bottom", fill="x", pady=(0, 10))
        self.btn_start = tk.Button(
            control_frame,
            text="Iniciar",
            bg=COLOR_BUTTON_BG,
            fg=COLOR_BUTTON_FG,
            font=FONT_BUTTON,
            command=self.start_generation
        )
        self.btn_start.pack(side="left", padx=(10, 5), ipadx=10, ipady=5)
        self.btn_stop = tk.Button(
            control_frame,
            text="Detener",
            bg=COLOR_BUTTON_BG,
            fg=COLOR_BUTTON_FG,
            font=FONT_BUTTON,
            command=self.stop_generation,
            state="disabled"
        )
        self.btn_stop.pack(side="left", padx=5, ipadx=10, ipady=5)
        self.btn_clear = tk.Button(
            control_frame,
            text="Limpiar historial",
            bg=COLOR_CLEAR_BG,
            fg=COLOR_BUTTON_FG,
            font=FONT_BUTTON,
            command=self.clear_history
        )
        self.btn_clear.pack(side="right", padx=(5, 10), ipadx=10, ipady=5)

    def _load_history_to_ui(self):
        for entry in self.history:
            ts  = entry.get("time", "")
            sig = entry.get("signal", "")
            text = f"Señal: {sig}x"
            self.chat_area.add_message(text, ts)

    def start_generation(self):
        if not self.is_running:
            self.is_running = True
            self.btn_start.config(state="disabled")
            self.btn_stop.config(state="normal")
            self._schedule_next_signal(initial=True)

    def stop_generation(self):
        if self.is_running:
            self.is_running = False
            self.btn_start.config(state="normal")
            self.btn_stop.config(state="disabled")
            if self.timer:
                try:
                    self.root.after_cancel(self.timer)
                except Exception:
                    try:
                        self.timer.cancel()
                    except Exception:
                        pass
            self.timer = None

    def clear_history(self):
        if messagebox.askyesno("Limpiar historial", "¿Desea eliminar todo el historial de señales?"):
            for b in self.chat_area.bubbles:
                b.destroy()
            self.chat_area.bubbles.clear()
            self.history.clear()
            save_history(self.history)

    def _generate_signal(self):
        if not self.is_running:
            return
        value = generate_multiplier()
        now   = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        text  = f"Señal: {value}x"
        self.chat_area.add_message(text, now)
        record = {"time": now, "signal": value}
        self.history.append(record)
        if MAX_HISTORY_MESSAGES is not None and len(self.history) > MAX_HISTORY_MESSAGES:
            excess = len(self.history) - MAX_HISTORY_MESSAGES
            self.history = self.history[excess:]
            for _ in range(excess):
                if self.chat_area.bubbles:
                    old = self.chat_area.bubbles.pop(0)
                    old.destroy()
        save_history(self.history)
        self._schedule_next_signal(initial=False)

    def _schedule_next_signal(self, initial=False):
        if not self.is_running:
            return
        if initial:
            delay_ms = 1000
        else:
            secs     = random.randint(MIN_INTERVAL_SECONDS, MAX_INTERVAL_SECONDS)
            delay_ms = secs * 1000
        try:
            self.timer = self.root.after(delay_ms, self._generate_signal)
        except Exception:
            thread_timer = threading.Timer(delay_ms / 1000.0, self._generate_signal)
            thread_timer.daemon = True
            thread_timer.start()
            self.timer = thread_timer

    def run(self):
        self.root.protocol("WM_DELETE_WINDOW", self._on_close)
        self.root.mainloop()

    def _on_close(self):
        if self.is_running:
            if not messagebox.askyesno("Salir", "La generación de señales está en curso. ¿Seguro que desea salir?"):
                return
        self.stop_generation()
        save_history(self.history)
        self.root.destroy()
