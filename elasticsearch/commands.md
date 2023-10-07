# Elasticsearch

## 查看相关

## 索引相关

```bash
# 查看索引
curl 'https://vpc-bigdata-fvvjej7abxknpd47qzoizurorm.ap-northeast-1.es.amazonaws.com/_cat/indices'?v
# 关闭写保护
curl -XPUT 'https://vpc-bigdata-fvvjej7abxknpd47qzoizurorm.ap-northeast-1.es.amazonaws.com/trade_settlement_detail/_settings' -H "Content-Type: Application/json" -d '{ "index": { "blocks": { "write": "false" } } }'

# 删除索引
curl -XDELETE 'http://xxx:9200/INDEX_NAME'

# 清理指定索引
curl -XPOST "http://IP:9200/INDEX_NAME/_cache/clear"  # 清理指定索引
curl -XPOST "http://IP:9200/_cache/clear"             # 清理所有
curl -XPOST "http://IP:9200/*/_cache/clear" # 同上

```

## 节点相关

## 集群配置相关

