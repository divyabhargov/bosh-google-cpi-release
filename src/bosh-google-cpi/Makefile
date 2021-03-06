default: test

# Builds bosh-google-cpi for linux-amd64
build:
	go build -o out/cpi bosh-google-cpi/main

# Build cross-platform binaries
build-all:
	gox -output="out/cpi_{{.OS}}_{{.Arch}}" bosh-google-cpi/main

# Prepration for tests
get-deps:
	# Go lint tool
	go get github.com/golang/lint/golint

	# Simplify cross-compiling
	go get github.com/mitchellh/gox

	# Ginkgo and omega test tools
	go get github.com/onsi/ginkgo/ginkgo
	go get github.com/onsi/gomega

# Cleans up directory and source code with gofmt
clean:
	go clean ./...

# Run gofmt on all code
fmt:
	gofmt -l -w .

# Run linter with non-stric checking
lint:
	@echo ls -d */ | grep -v vendor | xargs -L 1 golint
	ls -d */ | grep -v vendor | xargs -L 1 golint

# Vet code
vet:
	go tool vet $$(ls -d */ | grep -v vendor)

# Runs the unit tests with coverage
test: get-deps clean fmt lint vet build
	ginkgo -r -race -skipPackage=integration .

# Runs the integration tests from Concourse
testintci: get-deps
	ginkgo integration -slowSpecThreshold=500 -progress -nodes=3 -randomizeAllSpecs -randomizeSuites $(GINKGO_ARGS) -v

# Runs the integration tests with coverage
testint: check-proj get-deps clean fmt
	$(eval INTEGRATION_ADDRESS = $(shell gcloud --project=$(GOOGLE_PROJECT) compute addresses describe cfintegration --region=us-central1 | head  -n1 | cut -f2 -d' '))
    
	CPI_ASYNC_DELETE=true STEMCELL_URL=https://storage.googleapis.com/bosh-cpi-artifacts/bosh-stemcell-3262.12-google-kvm-ubuntu-trusty-go_agent-raw.tar.gz SERVICE_ACCOUNT=cfintegration@$(GOOGLE_PROJECT).iam.gserviceaccount.com GOOGLE_PROJECT=$(GOOGLE_PROJECT) EXTERNAL_STATIC_IP=$(INTEGRATION_ADDRESS) ginkgo integration -slowSpecThreshold=500 -progress -nodes=3 -randomizeAllSpecs -randomizeSuites $(GINKGO_ARGS) -v

# Checks and creates, if necessary, resources in a project required to run integration tests.
configint: check-proj
	@gcloud --project=$(GOOGLE_PROJECT) iam service-accounts describe cfintegration@$(GOOGLE_PROJECT).iam.gserviceaccount.com &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating service account"; \
		gcloud --project=$(GOOGLE_PROJECT) iam service-accounts create cfintegration; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute networks describe cfintegration &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating network cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute networks create cfintegration; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute networks describe cfintegration-custom &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating network cfintegration-custom"; \
		gcloud --project=$(GOOGLE_PROJECT) compute networks create cfintegration-custom --mode=custom; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute networks subnets describe cfintegration-custom-us-central1 --region=us-central1 &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating subnetwork cfintegration-custom-us-central1"; \
		gcloud --project=$(GOOGLE_PROJECT) compute networks subnets create cfintegration-custom-us-central1 --network=cfintegration-custom --range=192.168.0.0/16 --region=us-central1; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute addresses describe cfintegration --region us-central1 &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating static IP cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute addresses create cfintegration --region=us-central1; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute target-pools describe cfintegration --region us-central1 &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating target pool cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute target-pools create cfintegration --region=us-central1; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute target-pools describe cfintegration2 --region us-central1 &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating target pool cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute target-pools create cfintegration2 --region=us-central1; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged describe cfintegration --zone us-central1-a &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating instance group cfintegration in us-central1-a"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged create cfintegration --zone us-central1-a; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged describe cfintegration --zone us-central1-b &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating instance group cfintegration in us-central1-b"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged create cfintegration --zone us-central1-b; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute http-health-checks describe cfintegration &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating HTTP health check cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute http-health-checks create cfintegration; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute backend-services describe cfintegration &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating backend service cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute backend-services create "cfintegration" --http-health-checks "cfintegration" --port-name "http" --timeout "30"; \
        gcloud --project=$(GOOGLE_PROJECT) compute backend-services add-backend "cfintegration" --instance-group "cfintegration" --zone "us-central1-a" --balancing-mode "UTILIZATION" --capacity-scaler "1" --max-utilization "0.8"; \
        gcloud --project=$(GOOGLE_PROJECT) compute backend-services add-backend "cfintegration" --instance-group "cfintegration" --zone "us-central1-b" --balancing-mode "UTILIZATION" --capacity-scaler "1" --max-utilization "0.8"; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged describe cfintegration-ilb --zone us-central1-a &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating instance group cfintegration-ilb in us-central1-a"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged create cfintegration-ilb --zone us-central1-a; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged describe cfintegration-ilb --zone us-central1-b &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating instance group cfintegration-ilb in us-central1-b"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged create cfintegration-ilb --zone us-central1-b; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute health-checks describe cfintegration &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating TCP health check cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute health-checks create tcp cfintegration; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instances describe cfintegration-ilb-a --zone us-central1-a &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating instance cfintegration-ilb-a"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instances create cfintegration-ilb-a --zone us-central1-a --network cfintegration; \
		echo "Adding cfintegration-ilb-a to instance group cfintegration-ilb"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged add-instances cfintegration-ilb --instances cfintegration-ilb-a --zone us-central1-a; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instances describe cfintegration-ilb-b --zone us-central1-b &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating instance cfintegration-ilb-b"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instances create cfintegration-ilb-b --zone us-central1-b --network cfintegration; \
		echo "Adding cfintegration-ilb-b to instance group cfintegration-ilb"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged add-instances cfintegration-ilb --instances cfintegration-ilb-b --zone us-central1-b; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute backend-services describe cfintegration --region us-central1 &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Creating region backend service cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute backend-services create "cfintegration" --region us-central1 --health-checks "cfintegration" --protocol TCP --load-balancing-scheme INTERNAL --timeout "30"; \
        gcloud --project=$(GOOGLE_PROJECT) compute backend-services add-backend "cfintegration" --instance-group "cfintegration-ilb" --zone "us-central1-a" --region us-central1; \
        gcloud --project=$(GOOGLE_PROJECT) compute backend-services add-backend "cfintegration" --instance-group "cfintegration-ilb" --zone "us-central1-b" --region us-central1; \
	fi

	@echo "Your project is ready for integration tests."

# Deletes the resources created by the configint target
cleanint: check-proj
	@gcloud --project=$(GOOGLE_PROJECT) --quiet iam service-accounts describe cfintegration@$(GOOGLE_PROJECT).iam.gserviceaccount.com &> /dev/null; if [ $$? -ne 0 ]; then \
		echo "Deleting service account"; \
		gcloud --project=$(GOOGLE_PROJECT) iam service-accounts delete cfintegration@$(GOOGLE_PROJECT).iam.gserviceaccount.com; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute networks subnets describe cfintegration-custom-us-central1 --region=us-central1 &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting subnetwork cfintegration-custom-us-central1"; \
		gcloud --project=$(GOOGLE_PROJECT) compute networks subnets delete cfintegration-custom-us-central1 --region=us-central1 --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute addresses describe cfintegration --region us-central1 &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting static IP cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute addresses delete cfintegration --region=us-central1 --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute target-pools describe cfintegration --region us-central1 &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting target pool cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute target-pools delete cfintegration --region=us-central1 --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute target-pools describe cfintegration2 --region us-central1 &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting target pool cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute target-pools delete cfintegration2 --region=us-central1 --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute backend-services describe cfintegration &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting backend service cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute backend-services delete "cfintegration" --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute http-health-checks describe cfintegration &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting HTTP health check cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute http-health-checks delete cfintegration --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute backend-services describe cfintegration --region us-central1 &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting region backend service cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute backend-services delete cfintegration --region us-central1 --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute health-checks describe cfintegration &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting TCP health check cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute health-checks delete cfintegration --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instances describe cfintegration-ilb-a --zone us-central1-a &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting instance cfintegration-ilb-a in us-central1-a"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instances delete cfintegration-ilb-a --zone us-central1-a --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instances describe cfintegration-ilb-b --zone us-central1-b &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting instance cfintegration-ilb-b in us-central1-b"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instances delete cfintegration-ilb-b --zone us-central1-b --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged describe cfintegration-ilb --zone us-central1-a &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting instance group cfintegration-ilb in us-central1-a"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged delete cfintegration-ilb --zone us-central1-a --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged describe cfintegration-ilb --zone us-central1-b &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting instance group cfintegration-ilb in us-central1-b"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged delete cfintegration-ilb --zone us-central1-b --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute networks describe cfintegration &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting network cfintegration"; \
		gcloud --project=$(GOOGLE_PROJECT) compute networks delete cfintegration --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute networks describe cfintegration-custom &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting network cfintegration-custom"; \
		gcloud --project=$(GOOGLE_PROJECT) compute networks delete cfintegration-custom --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged describe cfintegration --zone us-central1-a &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting instance group cfintegration in us-central1-a"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged delete cfintegration --zone us-central1-a --quiet; \
	fi

	@gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged describe cfintegration --zone us-central1-b &> /dev/null; if [ $$? -eq 0 ]; then \
		echo "Deleting instance group cfintegration in us-central1-b"; \
		gcloud --project=$(GOOGLE_PROJECT) compute instance-groups unmanaged delete cfintegration --zone us-central1-b --quiet; \
	fi

	@echo "All integration tests resources have been removed. You will need to run the configint target before doing integration tests again."

check-proj:
	@if test "$(GOOGLE_PROJECT)" = "" ; then \
	  echo "GOOGLE_PROJECT must be set"; \
	  exit 1; \
	fi

check-destroy:
	@if test "$(DESTROY_OK)" = "" ; then \
	  echo "This is a potentially destructive operation. Existing instances with the "integration-delete" tag will be deleted. You must set DESTROY_OK to acknowledge."; \
	  exit 1; \
	fi
