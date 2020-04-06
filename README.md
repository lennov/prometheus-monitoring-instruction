Все это делалось на minikube, для minishift может немного отличаться, но я старался не использовать специфичный API
базовая инструкция на хабре https://habr.com/ru/company/flant/blog/340728/ уже не совсем актуальна

запускаем миникуб
minikube start

Логинимся под админом
oc login -u system:admin

ставим prometheus-operator из stable репы
helm install my-prometheus-operator stable/prometheus-operator
здесь сперва идет имя нашей установки, раньше был флаг --name, но в текущей версии helm ее уже не понимает
опционально можно выбрать неймспейс, например monitoring

смотрим эвенты
kubectl get events

если в эвентах видим ошибку деплоя для prometheus-operator вот такую
Error creating: pods "prometheus-operator-6959fc68f-" is forbidden: unable to validate against any security context constraint:
[spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 65534: must be in the ranges: [1000070000, 1000079999]]

тогда руками правим деплоймент
kubectl edit deploy/prometheus-operator
runAsUser: 1000070000 - тут конкретное значение нужно брать из ошибки, диапазон значений может быть разным

проверяем, что депломент поднялся
kubectl get events

проверяем что CRD создались
kubectl get customresourcedefinitions

должно вернуть что-то типа того
NAME                                    CREATED AT
alertmanagers.monitoring.coreos.com     2020-04-05T16:04:23Z
podmonitors.monitoring.coreos.com       2020-04-05T16:04:23Z
prometheuses.monitoring.coreos.com      2020-04-05T16:04:23Z
prometheusrules.monitoring.coreos.com   2020-04-05T16:04:23Z
servicemonitors.monitoring.coreos.com   2020-04-05T16:04:23Z
thanosrulers.monitoring.coreos.com      2020-04-05T16:04:23Z

если нет - надо ставить CRD руками

Deploy
=========
опционально можно собрать образ из
https://github.com/lennov/prometheus-test-app.git
./gradlew clean build dockerBuildImage

Чтобы заимпортить docker образ своего приложения из локальной репы докера
eval $(minikube -p minikube docker-env)
теперь импортим в репу, доступную миникубу
docker load -i app4.tar

в своем деплойменте сделать поименованный порт и настройки по комментам
пример prom-deploy.yaml
применяем
kubectl apply -f prom-deploy.yaml

в своем сервисе сделать поименованный порт и настройки по комментам
пример prom-svc.yaml

для своего деплоймента создать файл и настройки по комментам
пример: prom-sm.yaml

проверяем

App
=========
прокинем порт приложения на хост
kubectl port-forward svc/prom 8080
проверим, что оно доступно, отдает хелсчек и работает эндпойнт метрик
curl -s http://localhost:8080
curl -s http://localhost:8080/actuator/health
curl -s http://localhost:8080/actuator/prometheus

Prometheus
=========
прокинем порт прометея на хост
kubectl port-forward prometheus-my-prometheus-operator-prometheus-0 9090

смотрим, что в таргетах есть наша апишка, если есть, пробуем взять какую-нибудь java-овую метрику, например jvm_memory_used_bytes, убеждаемся что по ней есть данные

Grafana
=========
прокинем порт графаны на хост
kubectl port-forward svc/my-prometheus-operator-grafana 3000:80
заходим в графану
чтобы получить пароль
kubectl get secret --namespace default my-prometheus-operator-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
чтобы получить пользователя
kubectl get secret --namespace default my-prometheus-operator-grafana -o jsonpath="{.data.admin-user}" | base64 --decode ; echo

графана уже должна быть настроена, добавляем свой дашборд через импорт, правим под себя
пример: grafana-dashboard.json

опционально можно поставить деплоймент и сервис в другой неймспейс, примеры в /prod-ns
