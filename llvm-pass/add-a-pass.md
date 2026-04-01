```
cat myfirstpass.cpp
// FuncNamePass.cpp
#include "llvm/Pass.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;
namespace {

struct FuncNamePass : PassInfoMixin<FuncNamePass> {
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
    errs().write_escaped(F.getName()) << '\n';
    return PreservedAnalyses::all();
  }
};

}

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return {
    .APIVersion = LLVM_PLUGIN_API_VERSION,
    .PluginName = "FuncNamePass",
    .PluginVersion = LLVM_VERSION_STRING,
    .RegisterPassBuilderCallbacks = [](PassBuilder &PB) {
      PB.registerVectorizerStartEPCallback(
        [](llvm::FunctionPassManager &PM, OptimizationLevel Level) {
          PM.addPass(FuncNamePass());
        });
    }
  };
}

$ cat firstpass.sh
set -x

cat > demo.c << EOF
int main() {}
EOF
clang -emit-llvm demo.c -c -o demo.bc
clang -fpass-plugin=./FuncNamePass.so -O3 demo.c

opt -load FuncNamePass.so   -time-passes demo.bc
```
