# WebRTC file transfer
A WebRTC file transfer proof of concept, created as part of a school project

## Interesting Links
- a similar project: https://github.com/priyangsubanerjee/webrtc-file-transfer/blob/master/public/index.html
- discussion of webrtc security: https://webrtc-security.github.io/
- SDP explanation: https://webrtchacks.com/sdp-anatomy/
- MDN: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Connectivity
- MDN: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Perfect_negotiation
- Example on WebRTC website: https://github.com/webrtc/samples/blob/gh-pages/src/content/datachannel/filetransfer/js/main.js
- Signal blog about group calls: https://signal.org/blog/how-to-build-encrypted-group-calls/
- RFC8445 (ICE ... Interactive Connectivity Establishment): https://datatracker.ietf.org/doc/html/rfc8445

## Presentation Notes (in German 🇦🇹 as the presentation was held in German)
- Feature-Demo
  - verschicken einer Datei eben
- WebRTC ... Web Real-Time Communications
  - gedacht für Peer-to-Peer verbindungen für zB Video-Anrufe
  - auch Datenübertragung möglich
  - ich hab verwendet: Datenübertragung mit `DataChannels` (eigenes Protokoll)
  - WebRTC basiert auf einem Haufen von Protokollen
    - ICE, STUN, TURN, SDP, SCTP (Stream Control Transmission Protocol)
  - offener Standard
  - in die Wege geleitet von Google
  - Standards werden offen von W3C und IETF enwickelt
- Wie funktioniert das (vereinfacht):
  - die Schwierigkeit: P2P-Verbindungen zwischen Clients, die hinter NAT stehen
  - ich weiß ja meine öffentliche IP-Adresse nicht
  - deswegen gibt's STUN
  - Session Traversal Utilities for NAT
    - ich schick ein UDP-Paket an den Server und der sagt mir mit welcher IP er es empfangen hat, so ungefähr
  - TURN: Traversal Using Relays around NAT
    - ist dann eben ein Relay und schickt es von dem einen Client an den anderen
    - hat natürlich eine öffentliche statische IP
    - TURN-Server kann man selbst hosten (COTURN) oder dafür bezahlen
  - ICE: Interactive Connectivity Establishment
    - beschreibt den ganzen Prozess
    - Arten von Kandidaten:
    - host: Host (lokale IP-Adresse im zB Schulnetzwerk)
    - prflex: Peer Reflexive (festgestellt durch STUN-anfragen zwischen den Clients)
    - srflx: Server Reflexive (Adresse außerhalb des NAT)
    - relay: TURN Server (eben über einen TURN-Server)
  - <- *herzeigen des Beispiel-SDPs
- nicht so weit verbreitete Technologie
  - <- *Devtools herzeigen*
  - aber man kann durchaus nette Sachen damit bauen
    - Signal-Videoanrufe