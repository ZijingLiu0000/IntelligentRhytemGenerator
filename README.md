# IntelligentRhytemGenerator
import tkinter as tk
from tkinter import messagebox, simpledialog
import sounddevice as sd
import numpy as np
from mido import MidiFile, MidiTrack, Message
import time
import threading


class RhythmGenerator:
    def __init__(self, tempo=120):
        """
        Initializes the Rhythm Generator with a default tempo and rhythm patterns.
        """
        self.tempo = tempo
        self.rhythm_patterns = {
            "Electronic": [0.5, 0.5, 1, 1, 0.5, 0.5, 1, 1],
            "Jazz": [1, 0.75, 0.25, 1, 1.5, 0.5],
            "Rock": [1, 1, 1, 1, 0.5, 0.5, 1, 1]
        }
        self.sample_rate = 44100
        self.playing = False

    def generate_rhythm(self, style):
        """
        Generates the rhythm pattern based on the selected style.
        """
        pattern = self.rhythm_patterns.get(style, self.rhythm_patterns["Electronic"])
        return pattern

    def play_rhythm(self, rhythm_pattern):
        """
        Plays the rhythm pattern in real-time using sounddevice.
        """
        beat_duration = 60 / self.tempo
        self.playing = True
        for duration in rhythm_pattern:
            if not self.playing:
                break
            self.play_beat(duration * beat_duration)
            time.sleep(0.1)

    def play_beat(self, duration):
        """
        Generates and plays a single beat tone.
        """
        frequency = 440
        t = np.linspace(0, duration, int(self.sample_rate * duration), False)
        tone = 0.5 * np.sin(2 * np.pi * frequency * t)
        sd.play(tone, self.sample_rate)
        sd.wait()

    def save_to_midi(self, rhythm_pattern, filename="output_rhythm.mid"):
        """
        Saves the generated rhythm pattern to a MIDI file.
        """
        midi_file = MidiFile()
        track = MidiTrack()
        midi_file.tracks.append(track)

        beat_duration_ms = int((60 / self.tempo) * 1000)
        for duration in rhythm_pattern:
            track.append(Message('note_on', note=60, velocity=64, time=0))
            track.append(Message('note_off', note=60, velocity=64, time=int(duration * beat_duration_ms)))
        midi_file.save(filename)


class RhythmGeneratorApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Intelligent Rhythm Generator")
        self.root.geometry("450x450")
        self.root.configure(bg="#f0f0f5")

        self.generator = RhythmGenerator()
        self.current_rhythm_pattern = None
        self.play_thread = None
        self.realtime_preview = tk.BooleanVar(value=True)

        self.create_widgets()
        self.update_rhythm()

    def create_widgets(self):
        # Title Label
        title_label = tk.Label(self.root, text="Intelligent Rhythm Generator", font=("Helvetica", 18, "bold"),
                               bg="#f0f0f5", fg="#333")
        title_label.pack(pady=20)

        # Tempo Slider
        self.tempo_label = tk.Label(self.root, text="Tempo (BPM):", bg="#f0f0f5", font=("Helvetica", 12))
        self.tempo_label.pack()
        self.tempo_slider = tk.Scale(self.root, from_=60, to=180, orient=tk.HORIZONTAL, bg="#e0e0e0",
                                     troughcolor="#007aff", highlightthickness=0, command=self.on_tempo_change)
        self.tempo_slider.set(120)
        self.tempo_slider.pack(pady=10)

        # Rhythm Style Dropdown
        self.style_label = tk.Label(self.root, text="Rhythm Style:", bg="#f0f0f5", font=("Helvetica", 12))
        self.style_label.pack()
        self.style_var = tk.StringVar()
        self.style_menu = tk.OptionMenu(self.root, self.style_var, *self.generator.rhythm_patterns.keys(),
                                        command=self.on_style_change)
        self.style_var.set("Electronic")
        self.style_menu.pack(pady=5)

        # Real-Time Preview Toggle
        self.preview_checkbox = tk.Checkbutton(self.root, text="Enable Real-Time Preview",
                                               variable=self.realtime_preview, bg="#f0f0f5", font=("Helvetica", 12))
        self.preview_checkbox.pack(pady=10)

        # Play/Stop Button
        self.play_button = tk.Button(self.root, text="Start Real-Time Play", command=self.toggle_playback, bg="#007aff",
                                     fg="black", font=("Helvetica", 12, "bold"), relief="flat", padx=10, pady=5)
        self.play_button.pack(pady=10)

        # Save Button
        self.save_button = tk.Button(self.root, text="Save as MIDI", command=self.save_midi, bg="#34c759", fg="black",
                                     font=("Helvetica", 12, "bold"), relief="flat", padx=10, pady=5)
        self.save_button.pack(pady=5)

    def update_rhythm(self):
        """
        Updates the rhythm pattern and tempo based on user input.
        """
        style = self.style_var.get()
        self.generator.tempo = self.tempo_slider.get()
        self.current_rhythm_pattern = self.generator.generate_rhythm(style)

    def on_tempo_change(self, event):
        """
        Event handler for tempo slider change.
        """
        self.update_rhythm()
        if self.realtime_preview.get():
            self.restart_playback()

    def on_style_change(self, event):
        """
        Event handler for rhythm style change.
        """
        self.update_rhythm()
        if self.realtime_preview.get():
            self.restart_playback()

    def toggle_playback(self):
        """
        Toggles the playback state between play and stop.
        """
        if self.generator.playing:
            self.stop_playback()
            self.play_button.config(text="Start Real-Time Play")
        else:
            self.start_playback()
            self.play_button.config(text="Stop Real-Time Play")

    def start_playback(self):
        """
        Starts rhythm playback in a separate thread.
        """
        if self.play_thread and self.play_thread.is_alive():
            self.generator.playing = False
            self.play_thread.join()

        self.generator.playing = True
        self.play_thread = threading.Thread(target=self.generator.play_rhythm, args=(self.current_rhythm_pattern,))
        self.play_thread.start()

    def stop_playback(self):
        """
        Stops the current rhythm playback.
        """
        self.generator.playing = False
        if self.play_thread and self.play_thread.is_alive():
            self.play_thread.join()

    def restart_playback(self):
        """
        Restarts playback to reflect changes in tempo or rhythm style.
        """
        self.stop_playback()
        self.start_playback()

    def save_midi(self):
        """
        Saves the current rhythm as a MIDI file with a custom name.
        """
        self.update_rhythm()
        filename = simpledialog.askstring("Save as MIDI", "Enter file name (without extension):")
        if filename:
            filename = f"{filename}.mid"
            self.generator.save_to_midi(self.current_rhythm_pattern, filename)
            messagebox.showinfo("Saved", f"Rhythm saved as {filename}")


def main():
    root = tk.Tk()
    app = RhythmGeneratorApp(root)
    root.mainloop()


if __name__ == "__main__":
    main()
