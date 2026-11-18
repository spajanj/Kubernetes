# Kubernetes
Learning the Kubernetes
vehicle-api · nginx ingress · Helm
Header-based canary routing for ElectricBike and BIKE
One shared Helm chart, two values folders, two releases. ElectricBike is the default route. Requests carrying userType: BIKE are sent to BIKE.

Where does a request go?
No header
userType: BIKE
userType: ElectricBike
userType: BIKE
Client
ingress-nginx
/api/vehicle
ElectricBike service
main ingress
BIKE service
canary ingress
ElectricBike No header, so the default (main) ingress takes the request.
Design
Piece	ElectricBike release	BIKE release
Values file	vehicle-api/ElectricBike/values-<env>.yaml	vehicle-api/BIKE/values-<env>.yaml
userType	ElectricBike	BIKE
Deployment, HPA, secrets, config map	named from applicationName	same templates, different applicationName
Service	<ElectricBikeApplicationName>-ElectricBike-service	<BIKEApplicationName>-BIKE-service
Ingress	main, no canary annotations	canary, canary-by-header-value: "BIKE"
Two ingresses cover both cases. The canary ingress must use the same host and path as the main one, and nginx only honours it while the main ingress exists.

Changes to make
Nothing below has been applied to the repository yet.

1. Templates stay shared
vehicle-api/templates/service.yml
{{- if eq .Values.userType "ElectricBike" }}
#ElectricBike - SERVICE
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.ElectricBikeApplicationName }}-ElectricBike-service
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.ElectricBikeApplicationName }}
spec:
  selector:
    app: {{ .Values.ElectricBikeApplicationName }}
  type: ClusterIP
  ports:
    - name: appport
      port: {{ .Values.service.securePort }}
      targetPort: {{ .Values.service.httpPort }}
{{- end }}
{{- if eq .Values.userType "BIKE" }}
---
#BIKE - SERVICE
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.BIKEApplicationName }}-BIKE-service
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.BIKEApplicationName }}
spec:
  selector:
    app: {{ .Values.BIKEApplicationName }}
  type: ClusterIP
  ports:
    - name: appport
      port: {{ .Values.service.securePort }}
      targetPort: {{ .Values.service.httpPort }}
{{- end }}
vehicle-api/templates/ingress.yml
{{- if eq .Values.userType "ElectricBike" }}
#ElectricBike - MAIN (DEFAULT) INGRESS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.applicationName }}
  namespace: {{ .Values.namespace }}
  annotations:
    # keep the existing cert-manager, auth-*, proxy-* and snippet annotations unchanged
spec:
  rules:
    - host: {{ .Values.hostName }}
      http:
        paths:
          - path: /api/vehicle
            pathType: Prefix
            backend:
              service:
                name: {{ .Values.ElectricBikeApplicationName }}-ElectricBike-service
                port:
                  number: {{ .Values.service.securePort }}
{{- end }}
{{- if eq .Values.userType "BIKE" }}
---
#BIKE - CANARY INGRESS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.applicationName }}-BIKE-ingress
  namespace: {{ .Values.namespace }}
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "userType"
    nginx.ingress.kubernetes.io/canary-by-header-value: "BIKE"
spec:
  rules:
    - host: {{ .Values.hostName }}
      http:
        paths:
          - path: /api/vehicle
            pathType: Prefix
            backend:
              service:
                name: {{ .Values.BIKEApplicationName }}-BIKE-service
                port:
                  number: {{ .Values.service.securePort }}
{{- end }}
2. Values per variant
Add to every ElectricBike/values-{dev,int,uat,prod}.yaml (example: int):

userType: ElectricBike
applicationName: vehicle-api-int
ElectricBikeApplicationName: vehicle-api-int
Add to every BIKE/values-{dev,int,uat,prod}.yaml. The name must differ from ElectricBike, otherwise both releases fight over one Deployment:

userType: BIKE
applicationName: vehicle-api-BIKE-int
BIKEApplicationName: vehicle-api-BIKE-int
Services select pods by the label app: <applicationName>, which the Deployment template already sets, so each service reaches only its own pods.

Check before deploying BIKE. A new applicationName means a new service account (<name>-sa) and new secret names. If the AWS role in env.secret.iamRole trusts a specific service account name, add the BIKE one to its trust policy.
3. GitHub Actions
The existing deploy-bs-k8s.yml already picks a job from header_type. Two things to confirm in the reusable workflows (deploy-k8s-vehicle-api-ElectricBike.yml and deploy-k8s-vehicle-api-BIKE.yml in spa/workflows):

They read values from ElectricBike/ and BIKE/ respectively. If not, pass helm_values: ./vehicle-api/${{ github.event.inputs.header_type }}.
They use different Helm release names, for example vehicle-api-ElectricBike-<env> and vehicle-api-BIKE-<env>. A shared name makes the BIKE deploy overwrite ElectricBike.
Rollout order
Run the workflow with header_type = ElectricBike. This creates the main ingress and the ElectricBike service.
Run it again with header_type = BIKE. This adds the canary ingress and the BIKE service.
Verify:
kubectl get ingress -n spa-vehicle-int
curl -H "userType: BIKE" https://<hostName>/api/vehicle/...   # served by BIKE pods
curl https://<hostName>/api/vehicle/...                      # served by ElectricBike pods
Rollback BIKE by running helm uninstall for the BIKE release. All traffic returns to ElectricBike.
Behaviour to know
Header values are case sensitive. BIKE does not match BIKE and goes to ElectricBike.
Auth, buffer and snippet annotations on a canary ingress are ignored. The main ingress supplies them, so the canary only needs the three canary annotations.
Install BIKE before ElectricBike and the canary has no main ingress to attach to. Always deploy ElectricBike first in a new environment.
Ingress-nginx supports one canary ingress per path, which is why ElectricBike is the default and only BIKE is a canary.
Test checklist for the int rollout
Each behaviour above, with the command to run and the result to expect.

Behaviour	How to test	Expected
Case sensitivity	curl -H "userType: BIKE" https://<hostName>/api/vehicle/...
curl -H "userType: BIKE" https://<hostName>/api/vehicle/...	Lowercase reaches ElectricBike pods. Uppercase reaches BIKE pods. Tell them apart by pod name in the logs, or by a response header if the app sets one.
Auth, buffer and snippet inherited	Run kubectl describe ingress on both ingresses. Compare the generated nginx config in the controller pod.	BIKE requests still get the auth, buffer and snippet behaviour from the main ingress. An unauthenticated BIKE request returns 401 like ElectricBike.
Deploy order	In a scratch namespace, install BIKE first and send a request with userType: BIKE. Then install ElectricBike and repeat.	Before ElectricBike exists the canary does not route. After ElectricBike is installed, routing starts.
One canary per path	kubectl get ingress -n spa-vehicle-int -o yaml	Only the BIKE ingress carries canary annotations for /api/vehicle.
