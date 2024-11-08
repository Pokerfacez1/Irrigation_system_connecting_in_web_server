#include <SPI.h>
#include <Ethernet.h>
#include <DHT.h>

// Pin Definitions
#define DHTPIN 2         // Pin connected to DHT11 sensor
#define DHTTYPE DHT11    // DHT11 sensor type
#define SOIL_PIN A0      // Analog pin connected to Soil Moisture Sensor
#define RELAY_PIN 7      // Digital pin connected to Relay

// Initialize DHT sensor
DHT dht(DHTPIN, DHTTYPE);

// Ethernet Configuration
byte mac[] = { 0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED };  // MAC address (can be random)
EthernetServer server(80); // Web server on port 80

void setup() {
  Serial.begin(9600);
  
  // Start DHT sensor
  dht.begin();
  
  // Set pin modes
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Ensure relay is off by default
  
  // Start Ethernet (using DHCP)
  if (Ethernet.begin(mac) == 0) {
    Serial.println("Failed to configure Ethernet using DHCP");
    while (true); // Stop if DHCP fails
  }
  delay(1000);
  server.begin();
  
  // Print the assigned IP address
  Serial.print("Server is at ");
  Serial.println(Ethernet.localIP());
}

void loop() {
  // Listen for incoming clients
  EthernetClient client = server.available();
  if (client) {
    Serial.println("New client connected");
    boolean currentLineIsBlank = true;
    while (client.connected()) {
      if (client.available()) {
        char c = client.read();
        if (c == '\n' && currentLineIsBlank) {
          // Read sensor data
          float temperature = dht.readTemperature();
          float humidity = dht.readHumidity();
          int soilMoisture = analogRead(SOIL_PIN);
          bool relayState = digitalRead(RELAY_PIN);
          
          // Send HTML response
          client.println("HTTP/1.1 200 OK");
          client.println("Content-Type: text/html");
          client.println("Connection: close");
          client.println();
          
          // HTML content with line-by-line display
          client.println("<!DOCTYPE HTML>");
          client.println("<html>");
          client.println("<head>");
          client.println("<meta http-equiv='refresh' content='5'>"); // Refresh every 5 seconds
          client.println("<title>Arduino Sensor Monitor</title>");
          client.println("<style>");
          client.println("body { font-family: Arial, sans-serif; margin: 20px; line-height: 1.6; }");
          client.println(".button { display: inline-block; margin-top: 10px; padding: 10px; background-color: #007BFF; color: white; text-decoration: none; border-radius: 5px; }");
          client.println(".button:hover { background-color: #0056b3; }");
          client.println("</style>");
          client.println("</head>");
          client.println("<body>");
          client.println("<h2>Arduino Sensor Monitor</h2>");
          
          // Display Temperature
          if (isnan(temperature)) {
            client.println("<div><b>Temperature:</b> Error reading DHT11 sensor</div>");
          } else {
            client.println("<div><b>Temperature:</b> " + String(temperature) + " &#8451;</div>");
          }
          
          // Display Humidity
          if (isnan(humidity)) {
            client.println("<div><b>Humidity:</b> Error reading DHT11 sensor</div>");
          } else {
            client.println("<div><b>Humidity:</b> " + String(humidity) + " %</div>");
          }
          
          // Display Soil Moisture
          client.println("<div><b>Soil Moisture:</b> " + String(soilMoisture) + " (0-1023)</div>");
          
          // Display Relay Status
          client.println("<div><b>Relay Status:</b> " + String(relayState ? "ON" : "OFF") + "</div>");
          
          // Control Relay
          client.println("<div>");
          client.println("<a href='/?relay=on' class='button'>Turn Relay ON</a>");
          client.println("<a href='/?relay=off' class='button' style='background-color: #dc3545;'>Turn Relay OFF</a>");
          client.println("</div>");
          
          client.println("</body>");
          client.println("</html>");
          break;
        }
        
        // Check for newline
        if (c == '\n') {
          currentLineIsBlank = true;
        } else if (c != '\r') {
          currentLineIsBlank = false;
        }
      }
    }
    
    // Check for relay control commands
    if (client.available()) {
      String request = client.readString();
      if (request.indexOf("GET /?relay=on") != -1) {
        digitalWrite(RELAY_PIN, HIGH);
      } else if (request.indexOf("GET /?relay=off") != -1) {
        digitalWrite(RELAY_PIN, LOW);
      }
    }
    
    delay(1);
    client.stop();
    Serial.println("Client disconnected");
  }
}
