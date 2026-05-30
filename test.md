@startuml
skinparam handwritten false
skinparam monochrome false
skinparam shadowing false
skinparam defaultFontName "Arial"
skinparam ArrowColor #2C3E50
skinparam NoteBackgroundColor #FFF9E6
skinparam NoteBorderColor #F39C12
 
header Cloudflare One Secure Distributed Network
footer Distributed Infrastructure Deployment Topology
 
' Define Node Styles
skinparam node {
    BackgroundColor<<Cloudflare>> #FFF2CC
    BorderColor<<Cloudflare>> #D35400
    BackgroundColor<<AWS>> #EAF2F8
    BorderColor<<AWS>> #2980B9
    BackgroundColor<<OnPrem>> #EAFAF1
    BorderColor<<OnPrem>> #27AE60
    BackgroundColor<<Worker>> #F4ECF7
    BorderColor<<Worker>> #8E44AD
}
 
' -------------------------------------------------------------
' CLOUDFLARE EDGE (The Core Overlaid Secure Network Fabric)
' -------------------------------------------------------------
node "Cloudflare Global Edge Network" <<Cloudflare>> as cf_edge {
    component "Cloudflare Access\n(Identity & Auth Verification)" as cf_auth
    component "Cloudflare Gateway\n(Zero Trust & Device Posture)" as cf_gateway
    component "Cloudflare Edge Router\n(Virtual Private Routing Fabric)" as cf_router
    
    cf_auth -[hidden]down-> cf_gateway
    cf_gateway -[hidden]down-> cf_router
}
 
' -------------------------------------------------------------
' REMOTE WORKERS (Distributed Endpoints)
' -------------------------------------------------------------
node "Remote Worker Devices\n(Up to 20 Users)" <<Worker>> as remote_users {
    node "Mac / PC / Linux Laptop" as laptop {
        component "WARP Client" as warp_pc
        component "Native Terminal / SSH / RDP Client" as apps_pc
        apps_pc .> warp_pc : Encapsulates Traffic
    }
    
    node "iOS Device (iPad / iPhone)" as mobile {
        component "WARP Mobile Client" as warp_ios
        component "Safari Browser / App\n(Blink Shell, MS RDP)" as apps_ios
        apps_ios .> warp_ios : Encapsulates Traffic
    }
}
 
' -------------------------------------------------------------
' AMAZON WEB SERVICES (Cloud Infrastructure)
' -------------------------------------------------------------
node "Amazon Web Services (AWS)" <<AWS>> as aws_cloud {
    node "VPC (Private Subnet: 10.0.0.0/16)" as aws_vpc {
        node "EC2 Instance" as ec2_tunnel {
            component "cloudflared\n(Tunnel Daemon)" as aws_daemon
        }
        
        node "Internal Linux Servers" as aws_servers {
            component "Production App / DB\n(Private IP Access Only)" as internal_services
        }
        
        aws_daemon -[hidden]right-> internal_services
    }
}
 
' -------------------------------------------------------------
' LOCAL NETWORK (On-Premises Infrastructure)
' -------------------------------------------------------------
node "Corporate Office / Home Lab" <<OnPrem>> as local_net {
    node "Local LAN (Subnet: 192.168.1.0/24)" as lan_net {
        node "Always-On Machine / Server" as local_host {
            component "cloudflared / Mesh\n(Tunnel & Route Daemon)" as local_daemon
        }
        
        node "Office Desktop Computers" as local_desktops {
            component "Windows PC\n(RDP Server - Port 3389)" as win_pc
            component "Linux / Mac\n(SSH Server - Port 22)" as nix_pc
        }
        
        local_daemon -[hidden]right-> local_desktops
    }
}
 
' -------------------------------------------------------------
' NETWORKING & INTERCONNECTIONS (Outbound Only Tunnels)
' -------------------------------------------------------------
 
' Workers to Cloudflare
warp_pc ----> cf_edge : 1. WireGuard Tunnel (UDP 2408)\nAuthenticates via IdP
warp_ios ---> cf_edge : 1. WireGuard Tunnel (UDP 2408)\nAuthenticates via IdP
 
' Infrastructure to Cloudflare (Strictly Outbound - No Open Ports)
aws_daemon ====> cf_router : 2. Persistent Outbound Tunnel\n(TCP/QUIC over 443)\nAdvertises 10.0.0.0/16
local_daemon ===> cf_router : 2. Persistent Outbound Tunnel\n(TCP/QUIC over 443)\nAdvertises 192.168.1.0/24
 
' Routing traffic from Edge down to endpoints inside private zones
cf_router ...> aws_daemon : 3. Proxy Traffic
aws_daemon ----> internal_services : Target Local Resource
 
cf_router ...> local_daemon : 3. Proxy Traffic
local_daemon ---> win_pc : Route RDP Session
local_daemon ---> nix_pc : Route SSH Session
 
' Optional direct browser-based path note
note right of cf_auth
  Workers using iPads can bypass the
  WARP client entirely by navigating to
  **://yourcompany.com** in Safari.
  Cloudflare renders SSH/RDP in-browser.
end note
 
@enduml
 