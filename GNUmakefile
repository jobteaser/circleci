
ORBS = $(wildcard orbs/**/orb.yml)

all: validate

validate:
	./utils/validate $(ORBS)

publish-dev:
	./utils/publish-dev $(ORBS)

publish-stable:
	./utils/publish-stable $(ORBS)

.PHONY: all validate
