import os
import json
import cv2
import numpy as np
from kivy.app import App
from kivy.uix.widget import Widget
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.button import Button
from kivy.uix.colorpicker import ColorPicker
from kivy.graphics import Line, Color
from kivy.uix.slider import Slider
from kivy.uix.label import Label
from kivy.uix.scrollview import ScrollView
from kivy.uix.switch import Switch
from kivy.uix.spinner import Spinner
from kivy.core.window import Window
from kivy.uix.image import Image as KivyImage
from kivy.uix.popup import Popup
from kivy.uix.gridlayout import GridLayout
from kivy.uix.scrollview import ScrollView
from PIL import Image
import requests

class PaintWidget(Widget):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.line_width = 2
        self.color = (0, 0, 0, 1)
        self.strokes = []
        self.opacity = 1.0
        self.visible = True

    def on_touch_down(self, touch):
        if not self.collide_point(*touch.pos):
            return
        with self.canvas:
            Color(*self.color, self.opacity)
            touch.ud["line"] = Line(points=(touch.x, touch.y), width=self.line_width)
            self.strokes.append(touch.ud["line"])

    def on_touch_move(self, touch):
        if "line" in touch.ud:
            touch.ud["line"].points += [touch.x, touch.y]

    def set_color(self, rgba):
        self.color = rgba

    def set_brush_size(self, value):
        self.line_width = value

    def clear_layer(self):
        self.canvas.clear()
        self.strokes = []

    def export_as_image(self, filename="layer.png"):
        self.export_to_png(filename)

    def set_opacity(self, opacity):
        self.opacity = opacity

    def toggle_visibility(self, visible):
        self.visible = visible
        self.canvas.opacity = 1.0 if visible else 0.0

class DrawingApp(App):
    def build(self):
        self.root_layout = BoxLayout(orientation='vertical')
        self.canvas_area = BoxLayout(size_hint=(1, 0.8))
        self.layer_manager = ScrollView(size_hint=(1, 0.2))

        self.layers = []
        self.active_layer_index = 0

        # Add 3 default layers
        for i in range(3):
            layer = PaintWidget(size=(Window.width, Window.height))
            self.layers.append(layer)
            self.canvas_area.add_widget(layer)

        self.active_layer = self.layers[self.active_layer_index]

        # Layer management (scrollable list)
        layer_controls = BoxLayout(orientation='vertical', size_hint_y=None)
        layer_controls.height = len(self.layers) * 80  # Adjust per layer height

        for i, layer in enumerate(self.layers):
            layer_widget = BoxLayout(orientation='horizontal', size_hint_y=None, height=40)
            layer_widget.add_widget(Label(text=f"Layer {i}", size_hint_x=None, width=80))
            toggle_btn = Switch(active=layer.visible)
            toggle_btn.bind(active=lambda instance, value, layer=layer: layer.toggle_visibility(value))
            layer_widget.add_widget(toggle_btn)
            opacity_slider = Slider(min=0, max=1, value=layer.opacity, step=0.05)
            opacity_slider.bind(value=lambda instance, value, layer=layer: layer.set_opacity(value))
            layer_widget.add_widget(opacity_slider)
            layer_controls.add_widget(layer_widget)

        self.layer_manager.add_widget(layer_controls)

        # Top control panel (color, brush size, etc.)
        controls = BoxLayout(size_hint_y=0.2, height=50)

        color_btn = Button(text="Color")
        color_btn.bind(on_release=self.show_color_picker)

        ai_btn = Button(text="AI Ink")
        ai_btn.bind(on_release=self.send_to_ai)

        save_btn = Button(text="Save")
        save_btn.bind(on_release=self.save_layers)

        clear_btn = Button(text="Clear Layer")
        clear_btn.bind(on_release=self.clear_active_layer)

        self.brush_slider = Slider(min=1, max=20, value=2)
        self.brush_slider.bind(value=self.change_brush_size)

        controls.add_widget(color_btn)
        controls.add_widget(Label(text="Brush Size"))
        controls.add_widget(self.brush_slider)
        controls.add_widget(clear_btn)
        controls.add_widget(ai_btn)
        controls.add_widget(save_btn)

        self.root_layout.add_widget(self.canvas_area)
        self.root_layout.add_widget(self.layer_manager)
        self.root_layout.add_widget(controls)

        return self.root_layout

    def show_color_picker(self, instance):
        picker = ColorPicker()
        picker.bind(color=self.set_color)
        self.canvas_area.add_widget(picker)

    def set_color(self, instance, color):
        self.active_layer.set_color(color)

    def change_brush_size(self, instance, value):
        self.active_layer.set_brush_size(value)

    def clear_active_layer(self, instance):
        self.active_layer.clear_layer()

    def send_to_ai(self, instance):
        from PIL import Image

        # Save and merge all layers
        merged = Image.new("RGBA", (self.canvas_area.width, self.canvas_area.height), (255, 255, 255, 0))
        for i, layer in enumerate(self.layers):
            if layer.visible:
                filename = f"layer_{i}.png"
                layer.export_as_image(filename)
                img = Image.open(filename).convert("RGBA")
                merged = Image.alpha_composite(merged, img)

        merged_path = "merged.png"
        merged.save(merged_path)

        # Use OpenCV for inking
        img = cv2.imread(merged_path)
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
        blur = cv2.GaussianBlur(gray, (5, 5), 0)
        inked = cv2.adaptiveThreshold(blur, 255, cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
                                      cv2.THRESH_BINARY_INV, 11, 2)
        cv2.imwrite("inked.png", inked)
        self.show_inked_preview("inked.png")

    def show_inked_preview(self, image_path):
        img = KivyImage(image_path)
        popup = Popup(title='AI Inked Preview',
                      content=img,
                      size_hint=(0.8, 0.8))
        popup.open()

    def save_layers(self, instance):
        # Save each layer as PNG and save metadata in a JSON file
        os.makedirs("saved", exist_ok=True)
        metadata = {"layers": []}

        for i, layer in enumerate(self.layers):
            filename = f"saved/layer_{i}.png"
            layer.export_as_image(filename)
            metadata["layers"].append({
                "name": f"Layer {i}",
                "visible": layer.visible,
                "opacity": layer.opacity
            })

        with open("saved/layers_metadata.json", "w") as f:
            json.dump(metadata, f)

        # Zip everything together (PNG files and metadata)
        import zipfile
        with zipfile.ZipFile("saved/my_project.zip", "w") as zipf:
            for filename in os.listdir("saved"):
                zipf.write(os.path.join("saved", filename), filename)

if __name__ == "__main__":
    DrawingApp().run()
# webdraw
