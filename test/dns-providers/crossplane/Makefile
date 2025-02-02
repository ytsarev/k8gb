.PHONY: crossplane-setup
crossplane-setup:
	k3d cluster create crossplane-for-k8gb-test
	helm repo add crossplane-stable https://charts.crossplane.io/stable
	helm repo update
	helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
	kubectl wait --for=condition=available --timeout=120s deployment/crossplane -n crossplane-system
	kubectl wait --for condition=established --timeout=60s crd/deploymentruntimeconfigs.pkg.crossplane.io
	kubectl apply -f azure-family-install.yaml
	kubectl wait --for=condition=healthy provider.pkg.crossplane.io/provider-family-azure --timeout=120s
	kubectl wait --for condition=established --timeout=60s crd/providerconfigs.azure.upbound.io
	kubectl apply -f azure-provider-config.yaml
	kubectl apply -f aws-family-install.yaml
	kubectl wait --for=condition=healthy provider.pkg.crossplane.io/provider-family-aws --timeout=120s
	kubectl wait --for condition=established --timeout=60s crd/providerconfigs.aws.upbound.io
	kubectl apply -f aws-provider-config.yaml

.PHONY: azure-credentials
azure-credentials:
	kubectl create secret generic azure-creds -n crossplane-system --from-literal=credentials="$$(cat $(FILE))" --dry-run=client -o yaml | kubectl apply -f -

.PHONY: azure-dns-zone
azure-dns-zone:
	kubectl apply -f azure-dns-zone.yaml

.PHONY: aws-credentials
aws-credentials:
	kubectl create secret generic aws-creds -n crossplane-system --from-literal=credentials="$$(cat $(FILE))" --dry-run=client -o yaml | kubectl apply -f -

.PHONY: aws-dns-zone
aws-dns-zone:
	kubectl apply -f aws-dns-zone.yaml


.PHONY: delete-crossplane-cluster
delete-crossplane-cluster:
	k3d cluster delete crossplane-for-k8gb-test
