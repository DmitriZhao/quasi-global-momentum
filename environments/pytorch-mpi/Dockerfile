FROM user/base

USER $NB_USER
WORKDIR $HOME

# install openMPI
RUN mkdir $HOME/.openmpi/ \
    && wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.4.tar.gz \
    && gunzip -c openmpi-4.1.4.tar.gz | tar xf - \
    && cd openmpi-4.1.4 \
    && ./configure --prefix=$HOME/.openmpi/ --with-cuda \
    && make all install

ENV PATH $HOME/.openmpi/bin:$PATH
ENV LD_LIBRARY_PATH $HOME/.openmpi/lib:$LD_LIBRARY_PATH

# install conda
ENV PYTHON_VERSION=3.8.12
ENV PYTHONUNBUFFERED=1 PYTHONFAULTHANDLER=1 PYTHONHASHSEED=0
RUN curl -fsSL -v -o ~/miniconda.sh -O https://mirrors.bfsu.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh && \
    bash ~/miniconda.sh -b -p $HOME/conda && \
    rm ~/miniconda.sh
COPY pytorch-mpi/.condarc $HOME/conda/.condarc
ENV PATH $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin:$PATH
ENV LD_LIBRARY_PATH /usr/local/lib64:$HOME/conda/envs/pytorch-py$PYTHON_VERSION/lib:$LD_LIBRARY_PATH
RUN $HOME/conda/bin/conda update -n base conda && \
    $HOME/conda/bin/conda create -y --name pytorch-py$PYTHON_VERSION python=$PYTHON_VERSION &&\
    $HOME/conda/bin/conda install -y --name pytorch-py$PYTHON_VERSION \
        conda-build cython typing typing_extensions \
        numpy pyyaml scipy sympy ipython mkl mkl-include scikit-learn protobuf networkx \
        astunparse ninja setuptools cmake cffi future six requests dataclasses && \
    $HOME/conda/bin/conda install -y --name pytorch-py$PYTHON_VERSION -c pytorch magma-cuda118 && \
    $HOME/conda/bin/conda install -y --name pytorch-py$PYTHON_VERSION -c conda-forge av ffmpeg tabulate python-blosc && \
    $HOME/conda/bin/conda install -y --name pytorch-py$PYTHON_VERSION -c anaconda pandas && \
    $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin/pip config set global.index-url https://mirrors.bfsu.edu.cn/pypi/web/simple && \
    $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin/pip install spacy && \
    $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin/python -m spacy download en_core_web_sm && \
    $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin/python -m spacy download de_core_news_sm && \
    $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin/pip install nltk==3.4.5 transformers==2.2.2 pytelegraf pymongo influxdb kubernetes jinja2 lmdb tensorboard_logger pyarrow msgpack msgpack_numpy mpi4py POT opencv-python && \
    $HOME/conda/bin/conda clean -ya

# install pytorch, torchvision, torchtext.
RUN git clone --recursive https://github.com/pytorch/pytorch && \
    cd pytorch && \
    git checkout 44dac51 && \
    git submodule sync && \
    git submodule update --init --recursive --jobs 0 && \
    TORCH_CUDA_ARCH_LIST="8.6;8.9" TORCH_NVCC_FLAGS="-Xfatbin -compress-all" \
    CMAKE_PREFIX_PATH="$(dirname $(which $HOME/conda/bin/conda))/../" \
    $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin/python setup.py install && \
    cd .. && rm -rf pytorch

RUN git clone https://github.com/pytorch/vision.git torchvision \
    && cd torchvision \
    && git checkout tags/v0.14.1 \
    && $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin/python setup.py install && \
    cd .. && rm -rf torchvision

RUN git clone https://github.com/pytorch/text torchtext \
    && cd torchtext \
    && git checkout tags/v0.10.0 \
    && git submodule update --init --recursive \
    && $HOME/conda/envs/pytorch-py$PYTHON_VERSION/bin/python setup.py clean install && \
    cd .. && rm -rf torchtext
