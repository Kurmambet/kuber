```bash
cd part2/00_DEVOPS_MAY_CRY_APP/01_ЧАСТЬ_1_КЛАСТЕР/01_1.1_hello_terraform
```

## 1

просто готовит каталог. инициализирует, скачиает провайдеры, модули

```bash
terraform init
```

## 2

проверяет связщанность кода (типы, синтаксис, сылки на переменные, аттрибуты ...), проверяет конфиг

```bash
terraform validate
```

## 3

по умолчанию файл называется terraform.tfvars, но у нас demo.tfvars
просот прогноз

```bash
terraform plan -var-file=env/demo.tfvars
```

## 4

```bash
terraform apply -var-file=env/demo.tfvars
```

## 4.1

проверяем состояние до тех пор, пока не увидим 'No changes. Your infrastructure matches the configuration.'

```bash
terraform plan -var-file=env/demo.tfvars
```

## 5

```bash
terraform destroy -var-file=env/demo.tfvars
```

# команды

посмотреть адреса объектов, которые terraform считает своими

```bash
terraform state list

terraform state show local_file.journal
```

count и for_each.
при удалении чего-то из реплик, созданных с count (из середины), тераформ пересоздает все так чтобы у нас была четкая последовательность индексов.
короче `лучше всегда использовать for_each`, он дает стабильные независимые имена через map этих же индексов.
