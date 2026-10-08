Research project using the CT-RATE chest CT dataset.

## Container

Built from the lab's recon-dev image. To start:

'''bash
docker compose -f docker-compose-ctrate.yml up -d
docker exec -it aoibhe-ct-rate bash
'''

## Data

CT-RATE lives at '/data/ct-rate' on AXIS03, mounted into the
container at '/workspace/data'. Not tracked in git.

Note: the original NIfTI files have an identity affine with no
spacing or orientation information. Use the 'valid_fixed' /
'train_fixed' folders from the Hugging Face repo instead.

## Layout

- 'docker-compose-ctrate.yml' — container definition