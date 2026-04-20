# hello_network


## ① まず「同じネットワーク」

```
docker network create --subnet 192.168.10.0/24 lab-net

docker run -dit --name pc1 --network lab-net --ip 192.168.10.11 ubuntu bash
docker run -dit --name pc2 --network lab-net --ip 192.168.10.12 ubuntu bash
```

中に入って (`docker exec -it pc1 bash`)

```
apt update
apt install -y iproute2 iputils-ping net-tools tcpdump arping

ping 192.168.10.12
arp -a
```

## ② わざと壊す

以下は Docker では出来ない
```
ip addr change 192.168.20.11/24 dev eth0
RTNETLINK answers: Operation not permitted
```

- ネットワークを外す
docker network create --subnet 192.168.20.0/24 lan2
docker network disconnect lab-net pc1
docker network connect --ip 192.168.20.11 lan2 pc1

## ②-1 Routerを入れて繋げてみる


docker run -dit --name router --privileged ubuntu bash
//docker network connect --ip 192.168.10.1 lab-net router
//docker network connect --ip 192.168.20.1 lan2 router


## ① router の中に入る

```
docker exec -it router bash
```
## ② IP forwarding をON

echo 1 > /proc/sys/net/ipv4/ip_forward

cat /proc/sys/net/ipv4/ip_forward


docker exec -it pc1 bash
ip route add default via 192.168.10.1



# 作り直し
```
docker rm -f pc1 pc2 router 2>/dev/null
docker network rm lab-net lan2 2>/dev/null
```

# 作り直し

```

## ネットワークを作る

docker network create --subnet 192.168.10.0/24 lab-net
docker network create --subnet 192.168.20.0/24 lan2

## pc1 / pc2 を NET_ADMIN 付きで作る

docker run -dit --name pc1 --cap-add=NET_ADMIN --network lab-net --ip 192.168.10.11 ubuntu bash
docker run -dit --name pc2 --cap-add=NET_ADMIN --network lan2 --ip 192.168.20.12 ubuntu bash

## router も作る

docker run -dit --name router --cap-add=NET_ADMIN --network lab-net --ip 192.168.10.254 ubuntu bash
docker network connect --ip 192.168.20.254 lan2 router

## router を有効化
docker exec -it router bash

apt update
apt install -y iproute2 iputils-ping net-tools tcpdump
echo 1 > /proc/sys/net/ipv4/ip_forward
ip a
cat /proc/sys/net/ipv4/ip_forward

## pc1 / pc2 側の route 設定

#### pc1


docker exec -it pc1 bash
apt update
apt install -y iproute2 iputils-ping net-tools
ip route add 192.168.20.0/24 via 192.168.10.254
ip route


#### pc2


docker exec -it pc2 bash
apt update
apt install -y iproute2 iputils-ping net-tools
ip route add 192.168.10.0/24 via 192.168.20.254
ip route


# テスト

docker exec -it pc1 bash
ping 192.168.20.12

docker exec -it pc2 bash
ping 192.168.10.11


```