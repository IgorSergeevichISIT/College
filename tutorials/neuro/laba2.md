## Лабораторная работа №2: "Создание системы распознавания рукописных цифр с графическим интерфейсом"

**Цель:**  
Разработать нейронную сеть для классификации рукописных цифр и создать интерактивное приложение, где пользователь может рисовать цифру и получать предсказание в реальном времени.

---

### Шаг 1: Установка необходимых библиотек
```bash
pip install tensorflow keras numpy pillow matplotlib tk
```
- **TensorFlow/Keras** — для создания нейронной сети.
- **Pillow** — обработка изображений.
- **Tkinter** — графический интерфейс (входит в стандартную библиотеку Python).

---

### Шаг 2: Подготовка данных (MNIST)
```python
from tensorflow.keras.datasets import mnist

# Загрузка данных
(X_train, y_train), (X_test, y_test) = mnist.load_data()

# Нормализация (0-1) и преобразование формы
X_train = X_train.reshape(-1, 28, 28, 1) / 255.0
X_test = X_test.reshape(-1, 28, 28, 1) / 255.0
```
- **Объяснение:**  
  Данные приводятся к формату (28x28x1), так как изображения — градации серого. Нормализация ускоряет обучение[3][7][9].

---

### Шаг 3: Создание модели
```python
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Conv2D, MaxPooling2D, Flatten, Dense, Dropout

model = Sequential([
    Conv2D(32, (3, 3), activation='relu', input_shape=(28, 28, 1)),  # Сверточный слой
    MaxPooling2D((2, 2)),                                            # Пулинг
    Conv2D(64, (3, 3), activation='relu'),
    MaxPooling2D((2, 2)),
    Flatten(),                                                       
    Dense(128, activation='relu'),
    Dropout(0.5),                                                    # Регуляризация
    Dense(10, activation='softmax')
])

model.compile(optimizer='adam', 
              loss='sparse_categorical_crossentropy', 
              metrics=['accuracy'])
```
- **Функции активации:**  
  - `relu` — для скрытых слоёв (устраняет линейность).  
  - `softmax` — на выходе для вероятностей классов[1][5][7].

---

### Шаг 4: Обучение модели
```python
history = model.fit(X_train, y_train, 
                    epochs=10, 
                    batch_size=64, 
                    validation_split=0.2)
model.save('digit_model.h5')  # Сохранение модели
```
- **Рекомендации:**  
  - Следить за графиками точности и потерь (переобучение → добавить Dropout).  
  - Тестовая точность должна быть > 98%[7][9][45].

---

### Шаг 5: Создание GUI для рисования
```python
import tkinter as tk
from PIL import ImageGrab, ImageOps
import numpy as np

class PaintApp:
    def __init__(self):
        self.window = tk.Tk()
        self.canvas = tk.Canvas(self.window, width=280, height=280, bg='white')
        self.canvas.pack()
        
        # Кнопки
        btn_predict = tk.Button(self.window, text="Распознать", command=self.predict)
        btn_clear = tk.Button(self.window, text="Очистить", command=self.clear)
        btn_predict.pack(side=tk.LEFT)
        btn_clear.pack(side=tk.RIGHT)

        # Настройка рисования
        self.canvas.bind("<B1-Motion>", self.draw)
        self.image = np.zeros((28, 28))  # Холст для данных

    def draw(self, event):
        x, y = event.x, event.y
        self.canvas.create_oval(x, y, x+15, y+15, fill='black', outline='black')
        # Сохранение пикселей (масштабирование координат)
        self.image[y//10, x//10] = 1.0  # 28x28 grid

    def clear(self):
        self.canvas.delete("all")
        self.image = np.zeros((28, 28))

    def predict(self):
        # Преобразование в формат MNIST (инверсия цветов)
        img = ImageOps.invert(Image.fromarray(self.image * 255).convert('L'))
        img = img.resize((28, 28))
        img_array = np.array(img) / 255.0
        prediction = model.predict(img_array.reshape(1, 28, 28, 1))
        digit = np.argmax(prediction)
        print(f"Предсказание: {digit}")

app = PaintApp()
app.window.mainloop()
```
- **Особенности:**  
  - Рисование мышью с сохранением в массив 28x28.  
  - Инверсия цветов (MNIST: белый фон → чёрные цифры)[7][37][42].

---

### Шаг 6: Тестирование
1. Нарисуйте цифру на холсте.
2. Нажмите **Распознать** → в консоли выведется результат.  
3. Пример работы:  
   Пример GUI

---

### Дополнительные задания:
1. **Визуализация уверенности модели:**
   ```python
   import matplotlib.pyplot as plt

   def predict(self):
       ...
       plt.bar(range(10), prediction[0])
       plt.title('Вероятности цифр')
       plt.show()
   ```
2. **Улучшение точности:**  
   - Добавить аугментацию данных (повороты, сдвиги)[9][31].  
   - Увеличить количество слоёв CNN[9][31].

---

**Ссылки для углубления:**  
- [Keras MNIST пример](https://keras.io/examples/vision/mnist_convnet/)  
- [Tkinter документация](https://docs.python.org/3/library/tkinter.html)

