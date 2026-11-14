FROM public.ecr.aws/d3j8x8q7/olympus-base-rust:latest

WORKDIR /app
COPY . .

ENV CARGO_TERM_COLOR=never

# The workspace declares rust-version 1.100.0, which is newer than the toolchain the
# base image ships, so the matching toolchain is installed beside it and made the default.
RUN rustup toolchain install 1.100.0 --profile minimal \
 && rustup default 1.100.0 \
 && git config --system safe.directory '*' \
 && if [ ! -d .git ]; then rm -rf .git && git init -q && git add -A && git -c user.name=build -c user.email=build@localhost commit -qm base; fi \
 && cargo fetch --locked \
 && cargo build --offline --locked -p mago-analyzer --tests \
 && chmod -R a+rwX /app /opt/cargo /opt/rustup

CMD ["/bin/bash"]
