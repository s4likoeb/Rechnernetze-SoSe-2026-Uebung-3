# Übung 03

## Aufgabe 1

Zum Vergleich wurden das UDP- und das TCP-Programm ausgeführt und der Verkehr mit Wireshark aufgezeichnet.

Bei UDP sieht man, dass die Nachrichten direkt als einzelne Datagramme gesendet werden. In meiner Aufzeichnung sieht man zum Beispiel folgende UDP-Pakete:

```text
1  0.000000000   127.0.0.1   127.0.0.1   UDP   47   54554 -> 6000   Len=5
2  0.553036876   127.0.0.1   127.0.0.1   UDP   46   54554 -> 6000   Len=4
3 16.470890466   127.0.0.1   127.0.0.1   UDP   43   54554 -> 6000   Len=1
```

Dabei sieht man Quelladresse, Zieladresse, Protokoll, Paketlänge, Quellport, Zielport und Länge der Nutzdaten. Vor den UDP-Paketen gibt es keinen Verbindungsaufbau. Die Daten werden also direkt an den Zielport gesendet.

Bei TCP sieht man dagegen zuerst den Verbindungsaufbau:

```text
5 70.995636294   127.0.0.1   127.0.0.1   TCP   74   54092 -> 6000   [SYN]       Seq=0 Win=65495 Len=0
6 70.995652960   127.0.0.1   127.0.0.1   TCP   74   6000 -> 54092   [SYN, ACK]  Seq=0 Ack=1 Win=65483 Len=0
7 70.995664461   127.0.0.1   127.0.0.1   TCP   66   54092 -> 6000   [ACK]       Seq=1 Ack=1 Win=65536 Len=0
```

Diese drei Pakete bilden den TCP-Three-Way-Handshake. Zuerst sendet der Client ein `SYN`, danach antwortet der Server mit `SYN, ACK`, und anschließend bestätigt der Client mit `ACK`.

Die eigentlichen Nachrichten erkennt man danach an Paketen mit Nutzdaten:

```text
8  73.490112715   127.0.0.1   127.0.0.1   TCP   72   54092 -> 6000   [PSH, ACK]   Seq=1 Ack=1 Win=65536 Len=6
10 75.402936584   127.0.0.1   127.0.0.1   TCP   71   54092 -> 6000   [PSH, ACK]   Seq=7 Ack=1 Win=65536 Len=5
```

Die Pakete 9 und 11 sind Bestätigungen vom Server:

```text
9  73.490159966   127.0.0.1   127.0.0.1   TCP   66   6000 -> 54092   [ACK]   Seq=1 Ack=7  Win=65536 Len=0
11 75.402987918   127.0.0.1   127.0.0.1   TCP   66   6000 -> 54092   [ACK]   Seq=1 Ack=12 Win=65536 Len=0
```

Am Ende wird die TCP-Verbindung wieder geschlossen:

```text
12 77.258763495   127.0.0.1   127.0.0.1   TCP   66   54092 -> 6000   [FIN, ACK]   Seq=12 Ack=1  Win=65536 Len=0
13 77.301643683   127.0.0.1   127.0.0.1   TCP   66   6000 -> 54092   [ACK]        Seq=1  Ack=13 Win=65536 Len=0
```

Der Hauptunterschied ist, dass TCP verbindungsorientiert arbeitet. Man sieht Verbindungsaufbau, Bestätigungen und Verbindungsabbau. UDP arbeitet dagegen verbindungslos und sendet die Datagramme direkt. Gemeinsam ist beiden Protokollen, dass sie IP-Adressen und Ports verwenden, um Daten an das richtige Programm zu senden.

## Aufgabe 2a

```java
package oxoo2a;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.net.SocketException;
import java.nio.charset.StandardCharsets;

public class Main {
    private static final int PACKET_SIZE = 4096;
    private static volatile boolean running = true;

    private static void fatal(String comment) {
        System.out.println(comment);
        System.exit(-1);
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            fatal("Usage: java oxoo2a.Main -l <Port> oder java oxoo2a.Main <IP> <Port>");
        }

        int port = Integer.parseInt(args[1]);

        if (args[0].equalsIgnoreCase("-l")) {
            startServer(port);
        } else {
            startClient(args[0], port);
        }
    }

    private static void startServer(int port) throws Exception {
        DatagramSocket socket = new DatagramSocket(port);
        byte[] buffer = new byte[PACKET_SIZE];

        while (true) {
            DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
            socket.receive(packet);

            String message = new String(packet.getData(), 0, packet.getLength(), StandardCharsets.UTF_8);
            System.out.println(message);

            String answer = "Nachricht empfangen!";
            byte[] answerData = answer.getBytes(StandardCharsets.UTF_8);
            DatagramPacket response = new DatagramPacket(answerData, answerData.length, packet.getAddress(), packet.getPort());
            socket.send(response);

            if (message.equalsIgnoreCase("stop")) {
                break;
            }
        }

        socket.close();
    }

    private static void startClient(String host, int port) throws Exception {
        InetAddress address = InetAddress.getByName(host);
        DatagramSocket socket = new DatagramSocket();

        Thread receiver = new Thread(() -> receive(socket));
        receiver.start();

        BufferedReader keyboard = new BufferedReader(new InputStreamReader(System.in, StandardCharsets.UTF_8));
        String input;

        while ((input = keyboard.readLine()) != null) {
            byte[] data = input.getBytes(StandardCharsets.UTF_8);
            DatagramPacket packet = new DatagramPacket(data, data.length, address, port);
            socket.send(packet);

            if (input.equalsIgnoreCase("stop")) {
                running = false;
                socket.close();
                break;
            }
        }
    }

    private static void receive(DatagramSocket socket) {
        byte[] buffer = new byte[PACKET_SIZE];

        while (running) {
            try {
                DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
                socket.receive(packet);

                String message = new String(packet.getData(), 0, packet.getLength(), StandardCharsets.UTF_8);
                System.out.println(message);
            } catch (SocketException e) {
                if (running) {
                    System.out.println("Socket wurde geschlossen.");
                }
            } catch (Exception e) {
                System.out.println("Fehler beim Empfangen: " + e.getMessage());
            }
        }
    }
}
```

## Aufgabe 2b

```java
package oxoo2a;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.net.SocketException;
import java.nio.charset.StandardCharsets;

public class Main {
    private static final int PACKET_SIZE = 4096;
    private static volatile boolean running = true;

    private static void fatal(String comment) {
        System.out.println(comment);
        System.exit(-1);
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            fatal("Usage: java oxoo2a.Main <eigener-Port>");
        }

        int ownPort = Integer.parseInt(args[0]);
        DatagramSocket socket = new DatagramSocket(ownPort);

        Thread receiver = new Thread(() -> receive(socket));
        receiver.start();

        BufferedReader keyboard = new BufferedReader(new InputStreamReader(System.in, StandardCharsets.UTF_8));
        String input;

        while ((input = keyboard.readLine()) != null) {
            if (input.equalsIgnoreCase("stop")) {
                running = false;
                socket.close();
                break;
            }

            if (!input.startsWith("send ")) {
                System.out.println("Format: send <Ziel-IP-Adresse> <Ziel-Port> <Nachricht>");
                continue;
            }

            String[] parts = input.split(" ", 4);

            if (parts.length < 4) {
                System.out.println("Format: send <Ziel-IP-Adresse> <Ziel-Port> <Nachricht>");
                continue;
            }

            try {
                InetAddress targetAddress = InetAddress.getByName(parts[1]);
                int targetPort = Integer.parseInt(parts[2]);
                String message = parts[3];

                byte[] data = message.getBytes(StandardCharsets.UTF_8);
                DatagramPacket packet = new DatagramPacket(data, data.length, targetAddress, targetPort);

                socket.send(packet);
            } catch (Exception e) {
                System.out.println("Nachricht konnte nicht gesendet werden: " + e.getMessage());
            }
        }
    }

    private static void receive(DatagramSocket socket) {
        byte[] buffer = new byte[PACKET_SIZE];

        while (running) {
            try {
                DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
                socket.receive(packet);

                String message = new String(packet.getData(), 0, packet.getLength(), StandardCharsets.UTF_8);
                System.out.println("Nachricht von " + packet.getAddress().getHostAddress() + ":" + packet.getPort() + ": " + message);
            } catch (SocketException e) {
                if (running) {
                    System.out.println("Socket wurde geschlossen.");
                }
            } catch (Exception e) {
                System.out.println("Fehler beim Empfangen: " + e.getMessage());
            }
        }
    }
}
```

## Aufgabe 3

```java
package oxoo2a;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.InetAddress;
import java.net.Socket;
import java.nio.charset.StandardCharsets;

public class Main {
    private static volatile boolean running = true;

    private static void fatal(String comment) {
        System.out.println(comment);
        System.exit(-1);
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            fatal("Usage: java oxoo2a.Main <Server-IP> <Server-Port>");
        }

        String serverHost = args[0];
        int serverPort = Integer.parseInt(args[1]);

        InetAddress serverAddress = InetAddress.getByName(serverHost);
        Socket socket = new Socket(serverAddress, serverPort);

        BufferedReader serverReader = new BufferedReader(new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
        PrintWriter serverWriter = new PrintWriter(new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true);

        Thread receiver = new Thread(() -> receive(serverReader));
        receiver.start();

        BufferedReader keyboard = new BufferedReader(new InputStreamReader(System.in, StandardCharsets.UTF_8));
        String input;

        while ((input = keyboard.readLine()) != null) {
            serverWriter.println(input);

            if (input.equalsIgnoreCase("stop") || input.equalsIgnoreCase("quit")) {
                running = false;
                socket.close();
                break;
            }
        }
    }

    private static void receive(BufferedReader serverReader) {
        try {
            String line;

            while (running && (line = serverReader.readLine()) != null) {
                System.out.println(line);
            }
        } catch (Exception e) {
            if (running) {
                System.out.println("Verbindung beendet.");
            }
        }

        running = false;
    }
}
```

## Aufgabe 4

Mit `register <Name>` wird ein neuer Client beim Server angemeldet. Der Server speichert den eingegebenen Namen zusammen mit der Verbindung des Clients. Dadurch kann der Server später erkennen, welche Verbindung zu welchem Namen gehört.

Wenn ein Client `send <Empfängername> <Nachricht>` eingibt, zerlegt der Server diese Eingabe in den Befehl, den Empfänger und die eigentliche Nachricht. Anschließend sucht der Server den Empfänger in seiner Clientliste. Wenn der Name existiert, wird die Nachricht über die passende Verbindung an diesen Client weitergeleitet. Falls der Empfänger nicht existiert, bekommt der Sender eine Fehlermeldung.

Damit mehrere Clients gleichzeitig mit dem Server verbunden sein können, wird für jeden Client ein eigener Thread gestartet. Dadurch kann der Server weiterhin neue Clients annehmen und gleichzeitig Nachrichten von bereits verbundenen Clients verarbeiten.

```java
package oxoo2a;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.StandardCharsets;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class Main {
    private static final Map<String, ClientHandler> clients = new ConcurrentHashMap<>();

    private static void fatal(String comment) {
        System.out.println(comment);
        System.exit(-1);
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            fatal("Usage: java oxoo2a.Main <Port>");
        }

        int port = Integer.parseInt(args[0]);
        ServerSocket serverSocket = new ServerSocket(port);

        while (true) {
            Socket clientSocket = serverSocket.accept();
            Thread thread = new Thread(new ClientHandler(clientSocket));
            thread.start();
        }
    }

    private static class ClientHandler implements Runnable {
        private final Socket socket;
        private BufferedReader reader;
        private PrintWriter writer;
        private String username;

        ClientHandler(Socket socket) {
            this.socket = socket;
        }

        public void run() {
            try {
                reader = new BufferedReader(new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
                writer = new PrintWriter(new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true);

                writer.println("Bitte registrieren mit: register <Name>");

                String line;

                while ((line = reader.readLine()) != null) {
                    if (line.equalsIgnoreCase("stop") || line.equalsIgnoreCase("quit")) {
                        writer.println("Verbindung wird beendet.");
                        break;
                    }

                    if (username == null) {
                        register(line);
                    } else {
                        handleCommand(line);
                    }
                }
            } catch (Exception e) {
                System.out.println("Client-Verbindung beendet.");
            } finally {
                closeConnection();
            }
        }

        private void register(String line) {
            if (!line.startsWith("register ")) {
                writer.println("Bitte zuerst registrieren mit: register <Name>");
                return;
            }

            String requestedName = line.substring("register ".length()).trim();

            if (requestedName.isEmpty() || requestedName.contains(" ")) {
                writer.println("Der Name darf nicht leer sein und keine Leerzeichen enthalten.");
                return;
            }

            ClientHandler existingClient = clients.putIfAbsent(requestedName, this);

            if (existingClient != null) {
                writer.println("Dieser Name ist bereits vergeben.");
                return;
            }

            username = requestedName;
            writer.println("Registriert als " + username);
            writer.println("Befehle: send <Empfängername> <Nachricht>, list, quit");
            sendClientList();
        }

        private void handleCommand(String line) {
            if (line.equalsIgnoreCase("list")) {
                sendClientList();
                return;
            }

            if (!line.startsWith("send ")) {
                writer.println("Unbekannter Befehl.");
                return;
            }

            String[] parts = line.split(" ", 3);

            if (parts.length < 3) {
                writer.println("Format: send <Empfängername> <Nachricht>");
                return;
            }

            String targetName = parts[1];
            String message = parts[2];
            ClientHandler targetClient = clients.get(targetName);

            if (targetClient == null) {
                writer.println("Client nicht gefunden: " + targetName);
                return;
            }

            targetClient.writer.println(username + ": " + message);
            writer.println("Gesendet an " + targetName + ": " + message);
        }

        private void sendClientList() {
            writer.println("Aktive Clients: " + String.join(", ", clients.keySet()));
        }

        private void closeConnection() {
            if (username != null) {
                clients.remove(username);
                System.out.println(username + " getrennt.");
            }

            try {
                socket.close();
            } catch (Exception ignored) {
            }
        }
    }
}
```

## Aufgabe 5

```text
Bits:       0    1    1    0    1    0    0    1    0    1    0    1    1    0
Code:      _|‾  ‾|_  ‾|_  _|‾  ‾|_  _|‾  _|‾  ‾|_  _|‾  ‾|_  _|‾  ‾|_  ‾|_  _|‾
```

