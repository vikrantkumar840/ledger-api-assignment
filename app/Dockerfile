FROM python:3.12-slim

RUN groupadd -g 10001 appgroup && useradd -u 10001 -g appgroup -M -s /usr/sbin/nologin appuser

WORKDIR /app
COPY requirements.txt .

# PyYAML 5.4.1's setup.py is incompatible with Cython 3.x (known upstream bug).
# A pip constraints file forces Cython<3 even inside pip's isolated build env.
RUN echo "Cython<3" > /tmp/build-constraints.txt
ENV PIP_CONSTRAINT=/tmp/build-constraints.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
RUN chown -R appuser:appgroup /app

USER 10001
EXPOSE 8080
CMD ["python", "app.py"]
