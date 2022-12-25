
Для постоянного ip-адреса добавила балансировщик - Network Load Balaner.

Создаются Network Load Balaner и группа виртуальных машин с помощью terraform

[Ссылка на репозиторий с конфигами Terraform](https://github.com/irushenka/terraform_configs)

Проверяем созданные виртуальные машины:

![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-25%2002-39-15.png)

Потом создаются Deployment и Service как и ранее, определяется порт как и ранее и этот порт добавляем вручную в обработчик на балансировщике.


Смотрим ip-адрес балансировщика:

![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-25%2002-34-42.png)


Открывает ip-адрес балансировщика и порт и проверяем, что все работает:

![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-25%2002-33-49.png)