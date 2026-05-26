const WebSocket = require('ws');
const net = require('net');
const http = require('http');

// Render'ın servisi ayakta tutması için basit bir HTTP yanıtı
const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Bombom Socket Proxy Aktif!\n');
});

const wss = new WebSocket.Server({ server });

wss.on('connection', (ws, req) => {
    // Ruffle'ın gönderdiği dinamik IP ve Portu URL'den yakala
    // Örn: wss://proxy.com/?host=213.250.134.87&port=8000
    const url = new URL(req.url, `http://${req.headers.host}`);
    const targetHost = url.searchParams.get('host') || '213.250.134.87'; 
    const targetPort = url.searchParams.get('port') || 8000;

    console.log(`Bağlanılıyor -> ${targetHost}:${targetPort}`);

    // Oyunun TCP sunucusuna bağlan
    const tcpSocket = new net.Socket();
    tcpSocket.connect(targetPort, targetHost, () => {
        console.log('Oyun sunucusuna TCP bağlantısı başarılı.');
    });

    // Ruffle'dan (Telefondan) gelen veriyi Oyun Sunucusuna ilet
    ws.on('message', (msg) => {
        tcpSocket.write(msg);
    });

    // Oyun Sunucusundan gelen veriyi Ruffle'a (Telefona) ilet
    tcpSocket.on('data', (data) => {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(data);
        }
    });

    // Bağlantı kopmaları durumunda iki tarafı da temizle
    ws.on('close', () => tcpSocket.destroy());
    tcpSocket.on('close', () => ws.close());
    tcpSocket.on('error', (err) => {
        console.error('TCP Hatası:', err.message);
        ws.close();
    });
});

// Render'ın atayacağı portu dinle
const PORT = process.env.PORT || 10000;
server.listen(PORT, () => {
    console.log(`Proxy ${PORT} portunda dinleniyor...`);
});