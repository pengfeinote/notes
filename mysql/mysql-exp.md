1: 尽量不要使用update_time作为业务字段，可以使用其他字段（backup_time）等作为业务字段，避免特殊情况更新时，更新到update_time
2: mysql单实例TPS最好在4k以内，6k是高压线
