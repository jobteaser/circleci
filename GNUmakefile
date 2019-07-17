
ORBS = $(wildcard orbs/**/orb.yml)

all: validate

validate:
	./utils/validate $(ORBS)

publish-dev:
	./utils/publish-dev $(ORBS)

.PHONY: all validate
