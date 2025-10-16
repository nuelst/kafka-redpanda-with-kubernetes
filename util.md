# criar um tópico com 12 partições e fator de replicação 3
rpk topic create pedidos --partitions 12 --replication 3

# aumentar o número de partições (ex.: de 12 para 24)
rpk topic alter-config pedidos --set partition_count=24
