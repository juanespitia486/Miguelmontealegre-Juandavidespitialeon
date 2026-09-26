# Miguelmontealegre-Juandavidespitialeon
**1** 
en el siguiente codigo simula dos servidores independientes ( A y B ) que actúan como una red descentralizada básica. Cada mensaje enviado se "empaqueta" en un bloque conectado por SHA-256. Cuando el servidor A recibe un mensaje, crea el bloque y le copia inmediatamente al servidor B para mantenerlos con la misma informacion

```
import hashlib
import time
import json
from flask import Flask, request, jsonify
import threading
import requests

# --- 1. CLASE BLOCKCHAIN ---
class Blockchain:
    def __init__(self):
        self.chain = []
        # Crear el bloque génesis (primer bloque)
        self.create_block(data="Bloque Genesis (Inicio)", previous_hash="0")

    def create_block(self, data, previous_hash):
        """¿Cómo se crea un bloque? Se agrupan los datos, el timestamp, 
           el hash anterior y se calcula un nuevo hash SHA-256."""
        block = {
            'index': len(self.chain) + 1,
            'timestamp': time.time(),
            'data': data,
            'previous_hash': previous_hash
        }
        block['hash'] = self.hash_block(block)
        self.chain.append(block)
        return block

    def hash_block(self, block):
        # Genera la huella digital criptográfica (SHA-256) del bloque
        block_string = json.dumps(block, sort_keys=True).encode()
        return hashlib.sha256(block_string).hexdigest()

    def get_last_block(self):
        return self.chain[-1]

    def add_block_externally(self, block):
        # Valida que el bloque encaje con la cadena actual antes de aceptarlo
        last_block = self.get_last_block()
        if last_block['hash'] == block['previous_hash']:
            self.chain.append(block)
            return True
        return False

# --- 2. CONFIGURACIÓN DE LOS SERVIDORES (NODO A y NODO B) ---
app_a = Flask("Servidor_A")
app_b = Flask("Servidor_B")

blockchain_a = Blockchain()
blockchain_b = Blockchain()

# Ruta para enviar mensaje al Servidor A
@app_a.route('/enviar', methods=['POST'])
def enviar_a():
    mensaje = request.json.get('mensaje')
    last_block = blockchain_a.get_last_block()
    nuevo_bloque = blockchain_a.create_block(mensaje, last_block['hash'])
    
    # Sincronizar automáticamente con el Servidor B
    try:
        requests.post('http://localhost:5001/recibir', json=nuevo_bloque)
    except:
        pass
        
    return jsonify({"status": "Mensaje guardado en A y propagado", "bloque": nuevo_bloque}), 201

@app_a.route('/cadena', methods=['GET'])
def cadena_a():
    return jsonify(blockchain_a.chain)

# Ruta para recibir el bloque en el Servidor B
@app_b.route('/recibir', methods=['POST'])
def recibir_b():
    bloque = request.json
    exito = blockchain_b.add_block_externally(bloque)
    if exito:
        return jsonify({"status": "Bloque aceptado por B"}), 201
    return jsonify({"status": "Rechazado"}), 400

@app_b.route('/cadena', methods=['GET'])
def cadena_b():
    return jsonify(blockchain_b.chain)

# --- 3. EJECUCIÓN SIMULADA ---
if __name__ == '__main__':
    # Levantar servidores en hilos separados
    threading.Thread(target=lambda: app_a.run(port=5000, debug=False, use_reloader=False), daemon=True).start()
    threading.Thread(target=lambda: app_b.run(port=5001, debug=False, use_reloader=False), daemon=True).start()
    
    time.sleep(1) # Esperar arranque

    # Prueba de envío
    print("Enviando mensaje...")
    requests.post('http://localhost:5000/enviar', json={"mensaje": "Hola mundo por Blockchain"})

    # Ver cadenas
    print("\nCadena Servidor A:")
    print(requests.get('http://localhost:5000/cadena').json())

    print("\nCadena Servidor B (Sincronizada):")
    print(requests.get('http://localhost:5001/cadena').json())
```
---
**¿Cómo se crea un bloque?**
crear un bloque es empaquetar la nueva información junto con el rastro del bloque anterior y sellarlo con una firma criptográfica (SHA-256) para que nadie lo pueda alterar sin que se note como en palabras mas sencillas es como escribir una receta y la primera pagina ponerle (5) y la siguiente hoja iniciarla con ese digito o caracter y asi sucesivamente con cada pagina
---
