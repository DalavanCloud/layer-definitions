# Copyright 2018 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

load("@base_images_docker//package_managers:download_pkgs.bzl", "download_pkgs")
load("@base_images_docker//util:run.bzl", "container_run_and_extract")
load("@bazel_skylib//:bzl_library.bzl", "bzl_library")
load("@bazel_toolchains//rules/container:docker_toolchains.bzl", "language_tool_layer", "toolchain_container")
load("@io_bazel_rules_docker//contrib:test.bzl", "container_test")
load(":revisions.bzl", "BAZEL_INSTALLER")

licenses(["notice"])  # Apache 2.0

package(default_visibility = ["//visibility:public"])

exports_files(glob(["*.yaml"]))

bzl_library(
    name = "lib",
    srcs = glob(["*.bzl"]),
)

download_pkgs(
    name = "bazel_debs",
    image_tar = "@ubuntu1604//image",
    packages = [
        "bash-completion",
        "binutils",
        "git",
        "patch",
        "unzip",
        "wget",
        "zip",
    ],
)

# Create intermediate images with the Bazel installer.
# Note that this image is not the Bazel container but an intermediate image
# that we use to create the Bazel layer.
toolchain_container(
    name = "bazel_installer_image",
    base = "@ubuntu1604//image",
    files = [
        ":bazel.bazelrc",
        "@bazel_installer//file",
    ],
    packages = [
        "unzip",
    ],
)

# Extract the Bazel binary.
container_run_and_extract(
    name = "bazel_tar",
    commands = [
        "mv /bazel.bazelrc /etc/bazel.bazelrc",
        "chmod +x /bazel-installer.sh",
        "/bazel-installer.sh",
        "tar cvf /bazel.tar --mtime='1970-01-01' /usr/local/lib/bazel/* /etc/bazel.bazelrc",
    ],
    extract_file = "/bazel.tar",
    image = ":bazel_installer_image.tar",
)

# Bazel language_tool_layer target.
# Note that we do not add other tools/compilers required by Bazel
# (e.g., CC compiler, JDK) here as we include them in the final container.
language_tool_layer(
    name = "bazel-ltl",
    base = "@ubuntu1604//image",
    installables_tar = "@ubuntu1604_bazel_debs//file",
    symlinks = {
        "/usr/local/bin/bazel": "/usr/local/lib/bazel/bin/bazel",
    },
    tars = [":bazel_tar/bazel.tar"],
)

# Used by tests only.
genrule(
    name = "generate_bazel_yaml",
    srcs = [":tests.yaml.tpl"],
    outs = ["tests.yaml"],
    cmd = "sed 's|{_BAZEL_REVISION}|" + BAZEL_INSTALLER.revision + "|' <$< >$@",
)

container_test(
    name = "test",
    configs = [
        ":tests.yaml",
    ],
    image = ":bazel-ltl",
    verbose = True,
)
