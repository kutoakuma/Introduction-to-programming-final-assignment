# Introduction-to-programming-final-assignment
## This project ‘file encryption and transfer’ is made wile keeping in mind that sue to the major advancement of information and technology many people use network for transferring and storing their data which may not be entirely safe without some protection. This file encryption system is designed to protect your data from unauthorized users on the internet by converting your sensitive data into unreadable format. This system reduces the chances of data leak and increases the information security
# Below are the codes for the file encryption system
## Structure.py
~~~
class FileMetadata:
    """
    Custom data structure to store file metadata.
    """

    def __init__(self, filename, filesize, filehash):
        self.filename = filename
        self.filesize = filesize
        self.filehash = filehash

    def display_info(self):
        return f"Filename: {self.filename}, Size: {self.filesize} bytes, SHA256: {self.filehash}"
~~~
## utils.py
~~~
import hashlib
import base64
from cryptography.fernet import Fernet

def generate_key_from_password(password: str) -> bytes:
    """
    Generates a Fernet key from user password using SHA256.
    """
    hashed_password = hashlib.sha256(password.encode()).digest()
    return base64.urlsafe_b64encode(hashed_password)

def calculate_file_hash(file_path: str) -> str:
    """
    Calculates SHA256 hash of a file.
    """
    sha256 = hashlib.sha256()

    with open(file_path, "rb") as f:
        while chunk := f.read(4096):
            sha256.update(chunk)

    return sha256.hexdigest()
~~~
## encryption.py
~~~
from cryptography.fernet import Fernet
from utils import generate_key_from_password, calculate_file_hash
from structures import FileMetadata

def encrypt_file(file_path: str, password: str):
    """
    Encrypts file using password-based key.
    """
    key = generate_key_from_password(password)
    fernet = Fernet(key)

    with open(file_path, "rb") as file:
        data = file.read()

    encrypted_data = fernet.encrypt(data)

    encrypted_path = file_path + ".enc"

    with open(encrypted_path, "wb") as file:
        file.write(encrypted_data)

    file_hash = calculate_file_hash(encrypted_path)
    metadata = FileMetadata(encrypted_path, len(encrypted_data), file_hash)

    return metadata


def decrypt_file(file_path: str, password: str):
    """
    Decrypts encrypted file.
    """
    key = generate_key_from_password(password)
    fernet = Fernet(key)

    with open(file_path, "rb") as file:
        encrypted_data = file.read()

    decrypted_data = fernet.decrypt(encrypted_data)

    original_path = file_path.replace(".enc", "")

    with open(original_path, "wb") as file:
        file.write(decrypted_data)

    return original_path
~~~
## gui.py
~~~
import tkinter as tk
from tkinter import filedialog, messagebox, simpledialog
from tkinter import ttk
from encryption import encrypt_file, decrypt_file
from transfer import send_file, receive_file
from logger import log_event
import threading

BG_COLOR = "#02466B"
BUTTON_COLOR = "#994721"
HIGHLIGHT_COLOR = "#E5F4D9"
TEXT_COLOR = "white"



file_path = ""

def browse_file():
    global file_path
    file_path = filedialog.askopenfilename()
    if file_path:
        file_label.config(text=file_path)


def update_progress(current, total):
    percent = (current / total) * 100
    progress_bar["value"] = percent
    root.update_idletasks()


def encrypt_action():
    if not file_path:
        messagebox.showerror("Error", "Select file first.")
        return

    password = password_entry.get()
    if not password:
        messagebox.showerror("Error", "Enter password.")
        return

    try:
        encrypt_file(file_path, password)
        log_event("File encrypted")
        status_label.config(text="Encryption complete.")
    except Exception as e:
        messagebox.showerror("Error", str(e))


def decrypt_action():
    if not file_path:
        messagebox.showerror("Error", "Select file first.")
        return

    password = password_entry.get()
    if not password:
        messagebox.showerror("Error", "Enter password.")
        return

    try:
        decrypt_file(file_path, password)
        log_event("File decrypted")
        status_label.config(text="Decryption complete.")
    except Exception as e:
        messagebox.showerror("Error", str(e))


def send_action():
    if not file_path:
        messagebox.showerror("Error", "Select file first.")
        return

    host = simpledialog.askstring("Receiver IP", "Enter Receiver IP:")
    if not host:
        return

    progress_bar["value"] = 0

    def send_thread():
        try:
            send_file(file_path, host, progress_callback=update_progress)
            log_event("File sent")
            messagebox.showinfo("Success", "File sent successfully.")
        except Exception as e:
            messagebox.showerror("Error", str(e))

    threading.Thread(target=send_thread).start()


def receive_action():
    progress_bar["value"] = 0

    def receive_thread():
        try:
            metadata = receive_file(progress_callback=update_progress)
            log_event("File received")
            messagebox.showinfo("Success", f"Received: {metadata.filename}")
        except Exception as e:
            messagebox.showerror("Error", str(e))

    threading.Thread(target=receive_thread).start()


# GUI Setup
root = tk.Tk()
root.title("Secure File Encryption & Transfer Tool")
root.geometry("550x500")
root.configure(bg=BG_COLOR)

tk.Label(root, text="Secure File Encryption & Transfer Tool",
         font=("Arial", 16, "bold"),
         bg=BG_COLOR, fg=HIGHLIGHT_COLOR).pack(pady=15)

tk.Button(root, text="Browse File", bg=BUTTON_COLOR, fg=TEXT_COLOR,
          width=20, command=browse_file).pack(pady=5)

file_label = tk.Label(root, text="No file selected",
                      bg=BG_COLOR, fg=TEXT_COLOR, wraplength=450)
file_label.pack()

tk.Label(root, text="Enter Password:",
         bg=BG_COLOR, fg=HIGHLIGHT_COLOR).pack(pady=10)

password_entry = tk.Entry(root, show="*", width=30)
password_entry.pack()

tk.Button(root, text="Encrypt File", bg=BUTTON_COLOR, fg=TEXT_COLOR,
          width=25, command=encrypt_action).pack(pady=6)

tk.Button(root, text="Decrypt File", bg=BUTTON_COLOR, fg=TEXT_COLOR,
          width=25, command=decrypt_action).pack(pady=6)

tk.Button(root, text="Send File", bg=BUTTON_COLOR, fg=TEXT_COLOR,
          width=25, command=send_action).pack(pady=6)

tk.Button(root, text="Receive File", bg=BUTTON_COLOR, fg=TEXT_COLOR,
          width=25, command=receive_action).pack(pady=6)

# 🎯 Progress Bar
progress_bar = ttk.Progressbar(root, orient="horizontal",
                                length=400, mode="determinate")
progress_bar.pack(pady=15)

status_label = tk.Label(root, text="",
                        bg=BG_COLOR, fg=HIGHLIGHT_COLOR)
status_label.pack(pady=5)

root.mainloop()
~~~
## transfer.py
~~~
import socket
import os
from structures import FileMetadata
from utils import calculate_file_hash

BUFFER_SIZE = 4096
SEPARATOR = "<SEPARATOR>"


def send_file(file_path: str, host: str, port: int = 5001, progress_callback=None):
    """
    Sends file over local network.
    """
    s = socket.socket()
    s.connect((host, port))

    filesize = os.path.getsize(file_path)
    s.send(f"{file_path}{SEPARATOR}{filesize}".encode())

    with open(file_path, "rb") as f:
        sent = 0
        while True:
            chunk = f.read(BUFFER_SIZE)
            if not chunk:
                break

            s.sendall(chunk)
            sent += len(chunk)

            if progress_callback:
                progress_callback(sent, filesize)

    s.close()


def receive_file(host="0.0.0.0", port=5001, progress_callback=None):
    """
    Receives file from sender.
    """
    s = socket.socket()
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  # 🔥 fixes error 10048
    s.bind((host, port))
    s.listen(1)

    client_socket, address = s.accept()

    received = client_socket.recv(BUFFER_SIZE).decode()
    file_path, filesize = received.split(SEPARATOR)
    filename = os.path.basename(file_path)
    filesize = int(filesize)

    with open(filename, "wb") as f:
        bytes_received = 0
        while bytes_received < filesize:
            chunk = client_socket.recv(BUFFER_SIZE)
            if not chunk:
                break

            f.write(chunk)
            bytes_received += len(chunk)

            if progress_callback:
                progress_callback(bytes_received, filesize)

    client_socket.close()
    s.close()

    file_hash = calculate_file_hash(filename)
    metadata = FileMetadata(filename, filesize, file_hash)

    return metadata
~~~
## logger.py
~~~
import logging

logging.basicConfig(
    filename="activity.log",
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s"
)

def log_event(message: str):
    logging.info(message)
~~~
