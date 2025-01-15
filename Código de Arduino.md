    #include <SPI.h>
    #include <MFRC522.h>
    #include <Servo.h>
    #include <Wire.h>
    #include <LiquidCrystal_I2C.h>

    #define SS_PIN 10  // Pin de selección del lector RFID
    #define RST_PIN 9  // Pin de reinicio del lector RFID
    #define SERVO_PIN 6 // Pin del servomotor

    MFRC522 mfrc522(SS_PIN, RST_PIN); // Crear una instancia del lector RFID
    Servo servo; // Crear una instancia del servomotor

    // Inicializar la pantalla LCD (dirección I2C, columnas, filas)
    LiquidCrystal_I2C lcd(0x27, 16, 2); 

    // UID autorizado (
    byte tarjetaMaestra[] = {0xA3, 0x6C, 0x26, 0x13}; 
    // Lista de tarjetas autorizadas
    byte tarjetasAutorizadas[10][4]; // Array para almacenar hasta 10 tarjetas autorizadas
    int numTarjetas = 0; // Contador de tarjetas autorizadas
    bool acceso = false; // Estado de acceso

    void setup() {
    Wire.begin();
    lcd.init(); // Inicializa la pantalla LCD
    lcd.backlight(); // Enciende la luz de fondo
    lcd.setCursor(0, 0); // Establece el cursor en la primera columna y fila

    Serial.begin(9600); // Iniciar comunicación serial
    Serial.println("Inicio del programa");
    SPI.begin(); // Iniciar bus SPI
    mfrc522.PCD_Init(); // Iniciar el lector RFID
    servo.attach(SERVO_PIN); // Conectar el servomotor
    servo.write(0); // Asegurarse que el servomotor esté cerrado inicialmente (puerta abierta)
    
    lcd.begin(16, 2); // Inicializar la pantalla LCD con 16 columnas y 2 filas
    lcd.backlight(); // Encender la luz de fondo de la LCD
    lcd.print("Sistema Iniciado"); // Mensaje inicial en LCD
    
    Serial.println("Sistema de control de acceso RFID");
    }

    void loop() {
    // Verificar si hay una tarjeta presente
    if (!mfrc522.PICC_IsNewCardPresent()) {
        return;
    }

    // Leer la tarjeta
    if (!mfrc522.PICC_ReadCardSerial()) {
        return;
    }

    // Comparar la tarjeta leída con la tarjeta maestra
    if (compararTarjetas(mfrc522.uid.uidByte, tarjetaMaestra)) {
        if (!acceso) {
            Serial.println("Tarjeta maestra detectada. Cerrando puerta...");
            lcd.clear();
            lcd.print("Maestra: Cerrando");
            servo.write(90); // Cerrar la puerta (90 grados)
            acceso = true; // Cambiar estado a cerrado
            
            unsigned long startTime = millis(); // Capturar el tiempo inicial
            Serial.println("Tiempo de registro permitido por 3 segundos...");
            lcd.clear();
            lcd.print("Registro: 3 seg");
            delay(1000); // Esperar 1 segundo para permitir retirar la tarjeta maestra
            
            bool registrado = false; // Bandera para comprobar si se registró una tarjeta
            
            while (millis() - startTime < 3000) { // Esperar 3 segundos para permitir registro
                if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
                    if (compararTarjetas(mfrc522.uid.uidByte, tarjetaMaestra)) {
                        Serial.println("Tarjeta maestra detectada. Abriendo puerta...");
                        lcd.clear();
                        lcd.print("Maestra: Abriendo");
                        servo.write(0); // Abrir la puerta (0 grados)
                        acceso = false; // Cambiar estado a abierto
                        break; // Salir del bucle si se detecta la tarjeta maestra
                    }

                    if (compararTarjetasConLista(mfrc522.uid.uidByte)) {
                        Serial.println("Tarjeta registrada detectada. Desautorizando...");
                        servo.write(90);
                        lcd.clear();
                        lcd.print("Desautorizando...");
                        desautorizarTarjeta(mfrc522.uid.uidByte); // Desautorizando la tarjeta registrada
                    } else {
                        if (!registrado) { 
                            if (registrarTarjeta(mfrc522.uid.uidByte)) {
                                Serial.println("Nueva llave registrada.");
                                lcd.clear();
                                lcd.print("Llave Registrada");
                            } else {
                                Serial.println("La tarjeta ya está registrada.");
                                lcd.clear();
                                lcd.print("Ya Registrada");
                                delay(1000);
                            }
                            registrado = true; 
                        }
                    }
                }
                delay(100); // Pequeña pausa para evitar lecturas continuas rápidas.
            }
            Serial.println("Tiempo de registro finalizado.");
            lcd.clear();
            lcd.print("Cerrado");
        } else {
            Serial.println("Tarjeta maestra detectada. Abriendo puerta...");
            lcd.clear();
            lcd.print("Maestra: Abriendo");
            servo.write(0); // Abrir la puerta (0 grados)
            acceso = false; // Cambiar estado a abierto
        }
        
    } else {
        if (!acceso) {
            if (compararTarjetasConLista(mfrc522.uid.uidByte)) {
                Serial.println("Acceso permitido. Cerrando puerta...");
                lcd.clear();
                lcd.print("Acceso: Cerrando");
                servo.write(90); // Cerrar la puerta (90 grados)
                acceso = true; // Cambiar estado a cerrado
            } else {
                Serial.println("Acceso denegado.");
                servo.write(90);
                lcd.clear();
                lcd.print("Acceso Denegado");
            }
        } else {
          if (compararTarjetasConLista(mfrc522.uid.uidByte)) {
          //este
            Serial.println("Acceso permitido. Abriendo puerta...");
            lcd.clear();
            lcd.print("Acceso: Abriendo");
            servo.write(0); // Abriendo la puerta (0 grados)
            acceso = false; 
         } else {
          Serial.println("Acceso denegado.");
          lcd.clear();
          lcd.print("Acceso Denegado");
         }
        }
    }

    mfrc522.PICC_HaltA(); // Detener la lectura de la tarjeta

    }

    bool compararTarjetas(byte *tarjetaLeida, byte *tarjetaAutorizada) {
    for (byte i = 0; i < 4; i++) {
        if (tarjetaLeida[i] != tarjetaAutorizada[i]) {
            return false; 
        }
    }
    return true; 
    }

    bool registrarTarjeta(byte *nuevaTarjeta) {
    for (int i = 0; i < numTarjetas; i++) {
        if (compararTarjetas(nuevaTarjeta, tarjetasAutorizadas[i])) {
            return false; 
        }
    }

    if (numTarjetas < 10) { 
        memcpy(tarjetasAutorizadas[numTarjetas], nuevaTarjeta, sizeof(tarjetasAutorizadas[numTarjetas]));
        numTarjetas++;
        return true;
    }
    
    return false; 
    }
 
    bool compararTarjetasConLista(byte *tarjetaLeida) {
    for (int i = 0; i < numTarjetas; i++) {
        if (compararTarjetas(tarjetaLeida, tarjetasAutorizadas[i])) {
            return true; 
        }
    }
    return false; 
    }

    void desautorizarTarjeta(byte *tarjeta) {
    for (int i = 0; i < numTarjetas; i++) {
        if (compararTarjetas(tarjeta, tarjetasAutorizadas[i])) {
            for (int j = i; j < numTarjetas - 1; j++) {
                memcpy(tarjetasAutorizadas[j], tarjetasAutorizadas[j + 1], sizeof(tarjetasAutorizadas[j]));
            }
            numTarjetas--; 
            Serial.println("La tarjeta ha sido desautorizada.");
            return;
        }
    }
    }

